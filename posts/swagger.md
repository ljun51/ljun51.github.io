---
layout: post
title: Swagger 介绍
description: Swagger是一款强大的、开发API接口的开源框架，它有一个强大的生态系统工具链，这些工具可以帮你设计、构建、文档化、调用你的RESTful接口。Swagger除了提供开源工具之外，也提供SwaggerHub这样的商业化解决方案。
image: 
categories: java
---
### Swagger是什么
Swagger是一款强大的、开发API接口的开源框架，它有一个强大的生态系统工具链，这些工具可以帮你设计、构建、文档化、调用你的RESTful接口。Swagger除了提供开源工具之外，也提供SwaggerHub这样的商业化解决方案。

### 工具和规范
要使用Swagger，需要了解Swagger的三个工具和规范。这三个工具可以一起使用，也可以只使用其中的一个。它们是：

* 设计用的Swagger Editor：设计新的或编辑现有的API，它的强大之处在于可以实时、简洁的呈现你定义的接口，实时反馈错误处理。
* 构建用的Swagger Codegen：快速的部署你定义的API，生成支持40多中语言的服务端或客户端代码，增加工作效率。
* 文档化的Swagger UI：定义可视化的API资源，并生成漂亮的，可在任何环境中托管的交互式文档，让最终用户轻松开始使用您的API。

Swagger规范：Swagger规范是这三个工具的核心。规范是创建RESTful API的合约，详细的定义人和机器之间可读的所有资源和操作。规范是基于OpenAPI规范（OAS），OAS以标准化RESTful接口的定义方式，在开放，透明和协作的社区中开发。

### 安装使用
本节的安装教程是官方文档的简化版本，更详细的安装可以参考[官方文档][2]。

#### Swagger Editor
Swagger Editor是一个开源编辑器，根据Swagger规范设计，定义和文档化RESTful API。 Swagger Editor的源代码可以在GitHub中找到。本文编辑的时候Swagger Editor的版本是2.10.*。

GitHub: [https://github.com/swagger-api/swagger-editor][5]

Swagger Editor 可以在任何Web浏览器中工作，并且可以在本地托管或从Web访问。这里是Web官方网站访问[地址][3]，下面是本地安装的方法：

1. 安装[NodeJS][1]，如果你本地没有安装；
2. 安装Swagger Editor依赖, 运行`npm start`。

按照这种方法安装的应用在访问的时候JS加载有点问题，当时也没有深究。如果你也碰到了相同的问题，可以按照下面的方法安装：

> * npm install -g http-server
> * wget https://github.com/swagger-api/swagger-editor/releases/download/v2.10.4/swagger-editor.zip
> * unzip swagger-editor.zip
> * http-server swagger-editor

从Docker安装的方法:

> * docker pull swaggerapi/swagger-editor
> * docker run -p 80:8080 swaggerapi/swagger-editor

启动之后的效果：![swagger picture][4]

#### Swagger Codegen
Swagger Codegen是一个开源代码生成器，可以直接从Swagger定义的RESTful API构建服务器桩和客户端SDK。Swagger Codegen的源代码可以在GitHub中找到。本文编辑的时候Swagger Codegen的版本是2.2.*。

GitHub: [https://github.com/swagger-api/swagger-codegen][6]

在安装Swagger Codegen之前你需要安装：
1. Java > 7或更高版本

通过Homebrew安装使用：

> * brew install swagger-codegen
> * swagger-codegen help
> * swagger-codegen config-help -l javascript
> * swagger-codegen generate -i http://petstore.swagger.io/v2/swagger.json -l javascript

使用wget命令：

> * wget https://oss.sonatype.org/content/repositories/releases/io/swagger/swagger-codegen-cli/2.2.1/swagger-codegen-cli-2.2.1.jar
> * java -jar swagger-codegen-cli-2.2.1.jar help
> * java -jar swagger-codegen-cli-2.2.1.jar config-help -l php
> * java -jar swagger-codegen generate -i http://petstore.swagger.io/v2/swagger.json -l java

![Swagger Codegen][10]

#### Swagger UI
Swagger UI是一个开源项目，可直接从API的Swagger规范直观呈现Swagger定义的API的文档。Swagger UI的源代码可以在GitHub中找到。本文编辑的时候Swagger UI的版本是2.2.*。

GitHub: [https://github.com/swagger-api/swagger-ui][7]

Swagger UI是从Swagger规范中定义的任何API自动生成的，并且可以在浏览器中查看。

没有必要安装，构建或重新编译Swagger UI。 Swagger UI可以直接从GitHub存储库使用。 您可以使用swagger-ui代码。 请按照以下说明开始使用Swagger UI.

* 前往Swagger UI工程的[Github仓库][7];
* 克隆或下载仓库的zip文件；
* 进入本机的Swagger UI工程目录；
* 打开`dist`目录；
* 在浏览器中打开`dist/index.html`文件；
* Swagger UI现在将在浏览器中使用Swagger Petstore的默认呈现。 Swagger Petstore的JSON规范可以在这里找到 - http://petstore.swagger.io/v2/swagger.json
*注意：请记住，要加载规范并执行UI的试用请求，您需要启用CORS*
* 可以在顶部导航栏上的输入框中输入任何现有规范的YAML或JSON路径。

<image style="width:80%;" src="https://raw.githubusercontent.com/swagger-api/swagger.io/wordpress/images/docs/swaggerui-navigation.PNG"/>

例如，在上面的图像中，JSON规范可以在http://petstore.swagger.io/v2/swagger.json中找到，其文档可以在Swagger UI中生成。

默认情况下，打开Swagger UI将呈现Swagger Petstore API。可以将此默认规格更改为所需的API。一般来说，要这样做，你需要传递一个构造函数的URL作为你想要的默认值。
```
window.swaggerUi = new SwaggerUi({
  url: url, // here goes correct url
  …
});
```

你也可以这样做,在Swagger UI中作为默认呈现的规范的URL可以作为查询参数传递，如下所示：

> {your_host} /dist/index.html?url= {specification_link}

其中{your_host}是Swagger UI的主机，{specification_link}是你希望作为默认URL的地址。在构造函数中，您可以传递一个spec参数（在JSON中），如下所示：
```
window.swaggerUi = new SwaggerUi({
  spec: {}, // the specification location ,
  validatorUrl: “” // the URL of the validator of the specification
  …
});
```

如果规范没有任何语法错误，Swagger UI将将API作为交互式文档呈现。

![Swagger UI][9]

### 参考资料
* http://swagger.io
	
[1]: https://nodejs.org/en/
[2]: http://swagger.io/docs
[3]: http://editor.swagger.io/#/
[4]: http://2434zd29misd3e4a4f1e73ki.wpengine.netdna-cdn.com/wp-content/uploads/2016/11/Swagger_Editor_Full-1.png
[5]: https://github.com/swagger-api/swagger-editor
[6]: https://github.com/swagger-api/swagger-codegen
[7]: https://github.com/swagger-api/swagger-ui
[8]: https://raw.githubusercontent.com/swagger-api/swagger.io/wordpress/images/docs/swaggerui-navigation.PNG
[9]: http://2434zd29misd3e4a4f1e73ki.wpengine.netdna-cdn.com/wp-content/uploads/2016/11/swagger-ui.png
[10]: http://2434zd29misd3e4a4f1e73ki.wpengine.netdna-cdn.com/wp-content/uploads/2016/11/swagger-codegen-feature.png
