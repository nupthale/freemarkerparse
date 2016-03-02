# 前后端分离开发工具

## 1.概览
* tomcat server + ftl.jar（@新奇@亚楠）
* FMPP/FMtoll
* freemarker.js
* gulp-freemarker
* ftl-server(@安然)
* nei

### 1.1 解决问题思路
![](https://camo.githubusercontent.com/060e115d5dac3859b7b89e4955caff9ae56ba6a6/687474703a2f2f68616974616f2e6e6f732e6e6574656173652e636f6d2f30383335363962666435356134643233623439383335303736373234653762342e706e67)

&emsp;&emsp;上图是ftl文件解析的过程，输入ftl模板和对应java对象，经过freemarker.jar包解析后，得到输出文本；这个是我们在前后端分离前，运行java web服务执行的过程；那么，我们想脱离java web端逻辑来执行这个过程，该如何实现？关键点在这个java对象，下面给出两种实现：

* java web容器定制，抛弃后端逻辑；（tomcat server + ftl.jar）

&emsp;&emsp;通常情况下， 我们的请求与ftl的实际路径是没有关联的，我们需要在controller中定义url与ftl实际路径的关系，这样我们就需要维护这个关系，不利于抽象复用；由于我们是本地测试开发环境，不需要通过对应的url访问页面，直接通过实际的路径去访问，这样只需要一个controller统一处理，将url中路径的部分解析出来，处理对应的ftl即可；

1. 实现一个java类， 拦截所有`*.ftl`请求，拦截到后解析url路径，找到对应的ftl模板文件让freemarker处理后返回给浏览器；
2. 每个页面的同步假数据通过assign在ftl模板中定义；

&emsp;&emsp;以tomcat server + ftl.jar为例， tomcat配置web.xml如下：将所有*.ftl交给com.cjx.ftl.Ftl这个类去处理，ftl.jar是自己写的一个包，里面的只实现了上面说的统一处理请求，解析路径的功能；
![](http://haitao.nos.netease.com/62cad3066fd34d58b3cedceb7a41e0a7.jpg)

```java
package com.cjx.ftl;

import com.cjx.ftl.CjxBeanBeanWrapper;
import freemarker.ext.servlet.HttpRequestHashModel;
import freemarker.ext.servlet.HttpRequestParametersHashModel;
import freemarker.ext.servlet.HttpSessionHashModel;
import freemarker.ext.servlet.ServletContextHashModel;
import freemarker.template.Configuration;
import freemarker.template.Template;
import freemarker.template.TemplateException;
import java.io.File;
import java.io.IOException;
import java.util.HashMap;
import java.util.Locale;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

public class Ftl extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private Configuration cfg = null;
    private static final String ATTR_APPLICATION_MODEL = ".freemarker.Application";
    private static final String ATTR_REQUEST_MODEL = ".freemarker.Request";
    private static final String ATTR_REQUEST_PARAMETERS_MODEL = ".freemarker.RequestParameters";
    public static final String KEY_APPLICATION = "Application";
    public static final String KEY_REQUEST_MODEL = "Request";
    public static final String KEY_SESSION_MODEL = "Session";
    public static final String KEY_REQUEST_PARAMETER_MODEL = "Parameters";

    public Ftl() {
        this.cfg = new Configuration();
    }

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doPost(request, response);
    }

    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        ServletContext servletContext = request.getServletContext();
        response.setContentType("text/html; charset=UTF-8");
        request.setCharacterEncoding("UTF-8");
        String fPath = request.getServletContext().getRealPath("");
        String fullPath = request.getServletPath();
        if(!"".equals(fullPath) && fullPath.endsWith("ftl")) {
            HashMap root = new HashMap();
            ServletContextHashModel servletContextModel = (ServletContextHashModel)servletContext.getAttribute(".freemarker.Application");
            CjxBeanBeanWrapper wrapper = new CjxBeanBeanWrapper(false);
            if(servletContextModel == null) {
                servletContextModel = new ServletContextHashModel(this, wrapper);
                servletContext.setAttribute(".freemarker.Application", servletContextModel);
            }

            root.put("Application", servletContextModel);
            HttpRequestHashModel requestModel = (HttpRequestHashModel)request.getAttribute(".freemarker.Request");
            if(requestModel == null || requestModel.getRequest() != request) {
                requestModel = new HttpRequestHashModel(request, response, wrapper);
                request.setAttribute(".freemarker.Request", requestModel);
            }

            root.put("Request", requestModel);
            HttpRequestParametersHashModel reqParametersModel = (HttpRequestParametersHashModel)request.getAttribute(".freemarker.RequestParameters");
            if(reqParametersModel == null || requestModel.getRequest() != request) {
                reqParametersModel = new HttpRequestParametersHashModel(request);
                request.setAttribute(".freemarker.RequestParameters", reqParametersModel);
            }

            root.put("Parameters", reqParametersModel);
            HttpSession session = request.getSession(true);
            if(session != null) {
                root.put("Session", new HttpSessionHashModel(session, wrapper));
            }

            root.put("request", request);
            root.put("response", response);
            root.put("session", request.getSession(true));
            root.put("base", request.getContextPath());
            this.cfg.setDirectoryForTemplateLoading(new File(fPath));
            this.cfg.setAutoFlush(true);
            this.cfg.setDefaultEncoding("UTF-8");
            this.cfg.setEncoding(Locale.CHINA, "UTF-8");
            fullPath = fullPath.substring(1);
            Template t = this.cfg.getTemplate(fullPath, Locale.CHINA);

            try {
                t.process(root, response.getWriter());
            } catch (TemplateException var17) {
                var17.printStackTrace();
            } finally {
                response.getWriter().close();
            }
        }

    }
}

```
* 不依赖于java web容器，将ftl依赖的同步数据提取出来；

&emsp;&emsp;再次回来最开始的ftl模板解析流程图，脱离了java web环境，数据就不能存放在java对象中，既然前后端分离了，我们最希望它能维护在json文件中，但是json文件，如何才能解析替换ftl模板中的freemarker变量？接下来，就引出了Fmpp工具，它可以替我们完成这个工作；

## [FMPP](http://fmpp.sourceforge.net/)
* freemarker文本解析工具
* [使用](http://nupthale.github.io/fmppdoc/)

&emsp;&emsp;[FMToll](https://github.com/ijse/FMtoll)有Fmpp功能类似，实现不同，FMToll没有文档，如果感兴趣，可以研究下源码；

## 基于Fmpp的工具

&emsp;&emsp;有了fmpp，我们就可以使用fmpp命令将ftl模板与json文件结合生成解析文件；但是在实际工程应用中，我们不可能手动对分布在各个目录下的ftl模板文件，执行解析命令；这时候就需要一些工具帮我们实现解析的自动化；

### [freemarker.js](https://github.com/ijse/freemarker.js)
&emsp;&emsp;只有一个方法render；

```javascript
var Freemarker = require('freemarker.js');
var fm = new Freemarker({
  viewRoot: '/template',
  options: {
    /** for fmpp */
  }
});

// Single template file
fm.render(tpl, dataObject, function(err, html, output) {
  //...
});

```
上面代码的内部实现：根据参数和dataObject生成config文件，对tpl路径模板执行fmpp -c configfile命令；

### [gulp-freemarker](https://github.com/ijse/gulp-freemarker)
&emsp;&emsp;上面的freemarker.js只是简单的使用node封装了fmpp，并没有实现批量处理，gulp-freemarker就是使用gulp实现批量执行解析任务；gulp-freemarker就是基于freemarker.js实现的gulp插件，可以方便集成其他gulp任务；

### [ftl-server](https://www.npmjs.com/package/ftl-server)
&emsp;&emsp;到目前为止，我们已经解决了ftl模板的批量解析处理；但前后端分离做到这里还不能满足我们的开发需要；
#### 特性

* 解析freemarker模板
* 静态资源服务
* mock请求
* 代理请求
* livereload
* weinre

### [nei](https://github.com/genify/nei)
&emsp;&emsp;上面的ftl-server解决了我们开发过程中的全部问题，但是实际开发前，我们要与服务端定义接口，并根据接口mock数据，这个过程，使用上面的工具都要手动完成，定义的接口也不容易在文档中管理，那么我们就需要一套接口管理服务，帮助我们管理接口，并根据接口自动化的帮我们生成需要mock的数据供我们使用；

#### nei实际使用中的问题
1. stringify不支持，基于fmpp的工具都有这个问题；ftl-server是基于fmtoll实现的，可以使用stringify；nei mock list的临时解决办法：
* 在ftl模板中重新assign对应的list，不使用生成的mock list数据；

* 把mock数据中的list改成对象字符串
```
"list2": "[{a:1,b:2}]"
```

* 扩展fmpp， fmpp可以通过[DataLoader](http://fmpp.sourceforge.net/dataloader.html)对ftl模板访问的数据类型进行扩展;
```
sourceRoot:.
outputRoot:./dist/
data: {
    "word": "你好World",
    "list": [{
        "a": 1,
        "b": 2
    }
 ],
 "list2": "[{a:1,b:2}]",
 JSON:com.test.fmpp.JSONFactory()
}
```

2. 针对具体工程， 需要对nei配置进行修改， 参见[haitao引入nei配置](http://note.youdao.com/groupshare/web/file.html?token=9A40BAC52A374F168FC9F83A4575E553&gid=12651257)