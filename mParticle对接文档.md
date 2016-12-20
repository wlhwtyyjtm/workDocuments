### 整体说明
-----
#### mParticle数据传输流程
- mParticle从用户处获取到数据
- 调用lambda function，将Json数据作为参数传入
- lambda function内根据事件类型获取想要的字段，整理格式，调用EC2上service接口
- EC2上service接口接收到数据，将数据记入日志中

> 上述流程中，mParticle从用户获取数据不需要关心，这里主要是两部分内容：接受mParticle调用的lambda function以及最终将数据记入日志的EC2 service

#### lambda function
与mParticle对接的最重要的部分就是实现一个供mParticle调用传入数据的lambda function，lambda function为aws提供的一个功能，aws lambda具体信息可以查看[aws lambda介绍][1]。

### 具体流程
-----
数据对接具体流程将分为以下几个步骤：

- aws环境
- lambda function实现
- lambda function部署
- EC2 service实现

#### aws环境
因为与mParticle对接使用lambda function + EC2 service的方式，因此需要配置aws环境，以下环境配置均根据官方文档[aws lambda介绍][1]进行配置。

##### 账户配置
使用email或mobile number申请aws account，该账户将作为aws的根账户，该账户可以使用aws中的所有服务，但是只需要为真正使用到的服务付费。根账户创建完毕后会有一个唯一的一串数字作为accountid即账户。

aws服务访问时需要提供凭证，aws不建议使用根账户对服务进行访问，推荐使用***AWS Identity and Access Management(IAM)***来访问服务。

创建IAM用户步骤如下，详细步骤见官方[创建管理员用户][2]文档：
- 进入aws控制台
- 设置用户组
- 创建用户并归入用户组
- 在用户管理界面，产生创建的IAM用户的访问密钥，该密钥在后续CLI配置中使用
- 使用创建的管理员用户登录控制台

IAM用户创建完毕登录控制台时，要求输入以下信息：
- 账户 Account：根账户的数字accountid
- 用户名 User Name：刚刚创建的IAM用户名
- 密码 Password：刚刚创建的IAM用户密码

##### 设置aws cli
账户创建完毕后需要配置aws cli，以便可以通过命令号访问aws服务，官方文档[设置AWS CLI][3]中有详细的说明。

aws cli设置流程如下，具体见官方文档，这里不再赘述：
- 下载AWS CLI
- 配置CLI
- 验证

这里说一下AWS CLI的配置，配置可以使用命令行、环境变量、配置文件等多种方式，本文采用的是centos下配置文件的方式，更多配置方式见[配置AWS CLI][4]。

AWS CLI配置文件位于`~/.aws`目录下，包含`~/.aws/config`和`~/.aws/credentials`两个文件，分别对应CLI配置文件和AWS证书文件。

###### config
配置文件保存AWS区域、输出格式等信息
```Ini
[default]
region = ap-cn-north-1
output = text

[profile adminuser]
region = ap-southeast-1
output = text
```
region区域为aws服务运行时使用的区域终端节点，为了降低数据延迟最好选用附近的区域节点来部署服务，如这里`adminuser`使用的是新加坡区域`ap-southeast-1`，详见[AWS区域和终端节点][5]说明。

###### credentials
证书文件则保存了访问aws的密钥
```Ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[adminuser]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

配置文件和证书文件都可以使用`[profile ***]`或`[***]`的方式来指定不同的配置，在AWS CLI调用时加入`--profile ***`参数来指定使用的配置、证书内容。
> 注意：证书文件中使用的命名格式与配置文件不同，不包含`profile`前缀。

##### 创建lambda函数
使用IAM用户登录aws控制台，选择lambda服务，可以跟着[创建HelloWorld Lambda][6]来创建一个测试的lambda function。

这里说一下同mParticle对接需要创建的lambda function。
- 选择创建lambda function，创建一个Blank Function，不使用任何blueprint。
- lambda function由mParticle进行调用，因此不需要设置triggers，直接next。
- 在configure页面，配置lambda function的名字、介绍以及环境（这里选择Java 8），上传实现的lambda function zip文件（这部分将在后面lambda function实现部分说明），配置一些传入lambda function的环境变量
- 最后是配置handler和role，handler指定lambda function的入口函数，格式为`package.class::functionName`，role为该function的执行权限，这里创建一个默认权限的role即可(?)
- 创建完成后可以在lambda function页面上传test来进行测试

#### lambda function实现
lambda function实现主要参照[mParticle firehose-sample][mParticle firehose sample]项目，lambda function处理逻辑如下：
- lambda function handler函数获取到输入数据，反序列化为Message对象
- 根据不同的类型调用不同的函数进行处理
- 返回response，一次调用结束

下文lambda function实现参考[mParticle wiki][mParticle wiki]，对应的java方法和变量参考[java api document][mParticle java api]。

##### 依赖
mParticle-java-sdk需要JDK 1.8环境，sample工程中使用gradle来进行依赖包管理，用到的依赖包如下：
``` 
com.amazonaws:aws-lambda-java-core:1.1.0
com.amazonaws:aws-lambda-java-events:1.1.0
com.mparticle:java-sdk:+
com.squareup.okhttp:okhttp:2.7.5
```
依赖包包含aws lambda function和mparticle java sdk，最后的okhttp用来创建http请求将数据传给EC2 service（这部分后面会进行说明）。mParticle提供的Sample工程的gradle配置见[build.gradle][mParticle sample build.gradle]。

##### 实现
Sample工程中已经完成了lambda function的创建，这里需要做的就是实现`MessageProcessor`，在示例工程中就是`SampleExtension`类。在`SampleExtension`类中需要进行如下四部分功能实现：
1. Module Registration Request and Response
2. Event Processing Request and Response
3. Audience Membership Subscription Request and Response
4. Audience Subscription Request and Response

由于本文对接没有使用到Audience相关数据，因此只需要完成1、2两部分即可。

###### Module Registration
mParticle wiki页面[Module Registration][mParticle Module Registration]详细说明了该部分的功能。

本文只对本次对接用到的部分进行说明，主要包括以下三部分：
- 指定支持的设备、用户标识，这里使用的是全部的标识，标识信息来自`DeviceIndentity.Type`和`UserIdentity.Type`
- 指定支持的事件类型，事件类型来自枚举类型`Event.Type`，本文使用到的事件类型包括`CUSTOM_EVENT`、`PRODUCT_ACTION`、`USER_ATTRIBUTE_CHANCE`、`APPLICATION_STATE_TRANSITION`四种
- 指定支持的RuntimeEnvironment，来自枚举类型`RuntimeEnvironment.Type`，此处使用到了`IOS`和`ANDROID`两种类型

生成response对象，并返回给mParticle，mParticle会根据response中的设置给lambda function传输对应的数据。

###### Event Processing
mParticle wiki页面[Event Processing][mParticle Event Processing]对这一部分进行了详细说明。

事件处理是根据支持的事件类型，重写对应的方法来自定义处理逻辑。

Event Type | Method
---|---
CUSTOM_EVENT | MessageProcessor#processCustomEvent
PRODUCT_ACTION | MessageProcessor#processProductActionEvent
USER_ATTRIBUTE_CHANCE | MessageProcessor#processUserAttributeChanceEvent
APPLICATION_STATE_TRANSITION | MessageProcessor#processApplicationStateTransitionEvent

重写方法后，根据[java api document][mParticle java api]中各个Event类包含的方法以及github上的[示例输入json][mParticle EventProcessingRequest.json]，获取到需要的数据字段，组织成指定的格式之后，调用EC2 service将数据保存下来。

获取应用名称示例如下，其余字段根据具体需求定义获取，此处不一一说明:
```Java
private String getGameidFromEvent(Event event) {
	RuntimeEnvironment environment = event.getContext().getRuntimeEnvironment();
	String gameid = "";
	if (environment.getType().equals(RuntimeEnvironment.Type.IOS)) {
		gameid = ((IosRuntimeEnvironment) environment).getApplicationName();
	} else if (environment.getType().equals(RuntimeEnvironment.Type.ANDROID)) {
		gameid = ((AndroidRuntimeEnvironment) environment).getApplicationName();
	}
	return gameid;
}
```

##### 调用接口写日志
获取到需要的数据后，根据需求将数据组织成特定格式并存储下来，这里使用逗号分隔将获取到数据组织成字符串的形式，通过调用EC2 service接口将数据写入日志中。

> 为什么使用接口接收数据：原需求是lambda function接收到mParticle数据后，组织成想要的格式写入AWS s3中。在阅读[aws s3][aws s3]说明及对应的[java api][aws s3client java api]后，发现java api使用AmazonS3Client来操纵s3和上传数据到Bucket，且上传数据不支持append的方式，对于本文需求显得不太合适，同时lambda function无法进行数据的缓存，因此采用EC2 service来接受数据的方式。

本文使用okhttp包来实现http接口的访问，该包的具体信息见[okhttp官网][okhttp home page]，这里只是构建简单的请求，将要传输的数据作为请求参数传递给service，示例实现代码如下：
```Java
OkHttpClient client = new OkHttpClient();
String url = "****"; // Your http request url with paramters here
Request request = new Request.Builder().url(url).build();
Response response = client.newCall(request).execute(); // response of the request
return response.body().toString();
```

#### lambda function部署
实现lambda function后可以编写一些测试代码对实现的lambda function和事件处理函数进行测试，测试使用`junit.framework.TestCase`的继承类来实现。

完成lambda function和测试代码的编写后，可以在命令行使用`gradle test`命令来对编写好的代码进行测试，可以采用带参数的命令`gradle test --info --stacktrace`来输出更多信息。

测试完成后，使用`gradle build`命令来将代码打包，若使用mParticle提供的示例工程，其配置文件`build.gradle`中有如下配置，打包之后会在`./build/distributions`目录下生成对应的zip文件，将该zip压缩包上传到aws lambda function即可。
```Nginx
//create the zip file by executing: ./gradlew build
build.dependsOn buildZip
```
在aws lambda function创建页面上传生成的zip代码包，即将lambda function部署到线上。

##### 环境变量设置
aws lambda function允许在管理页面添加、删除、修改环境变量，使得对于一些参数性的变更可以通过在页面修改环境变量实现而不需要重新打包上传整个lambda function，详细的说明见官方文档[]。

在function管理的code页面添加、删除、修改Environment variables，设置的环境变量同在windows或linux下设置的环境变量相同，在java代码中获取环境变量使用如下系统函数：
```Java
Map<String, String> System.getenv();    // 获取全部环境变量
String System.getenv(String name);      // 获取指定环境变量
```

##### 调用测试
lambda function创建完成之后可以手动输入一个event来进行测试。通过选择`Action -> Configure test event`来管理修改测试event，点击`Test`按钮即可使用输入的测试event来测试lambda function。

##### 为mParticle配置权限
创建部署好的lambda function由mParticle平台进行调用，因此需要为mParticle调用该function配置相应的权限。权限配置参照mParticle文档[Deployment][mParticle Deployment]页面。

在aws cli为对应的principal配置调用指定lambda function的权限，需要为lambda function创建别名来方便调用。

别名创建在function管理页面，通过选择`Actions -> Create alias`按钮来实现，创建function别名，并将别名指定到最新的版本`$LATEST`。

别名创建完毕后，在管理页面选择`Qualifiers -> Aliases`，选择刚刚创建的别名，此时页面右上角显示的ARN值即为配置权限是要使用到的function name，ARN值如下所示：
```
arn:aws:lambda:ap-southeast-1:87091*******:function:supercell-data-catcher:datacatcher
```
根据[Deployment][mParticle Deployment]页面说明，为mParticle配置权限即可。

#### EC2 service 实现
接受lambda function传输日志数据的接口，采用在EC2上部署基于Dropwizard开发的接口实现。EC2信息详见[官方文档][aws ec2]，这里使用的是最基础的免费版进行测试，不做过多介绍。

Dropwizard[文档地址][dropwizard document]，这里创建了一个最简单的接口，接受lambda function调用，将调用时传入的参数按照指定格式保存到一个日志文件中，日志文件按天切分。

Dropwizard接口实现不做说明，详见[Dropwizard 文档][dropwizard document]，这里只对日志的配置文件进行说明，对不同的日志级别（ERROR、INFO、DEBUG）输出到不同的日志文件中，接口接收到的传入的数据则应该保存在单独的诗句日志文件中，配置文件说明如下：
```YAML
# 日志配置
logging:
  # 默认日志级别
  level: DEBUG
  
  # 设定ERROR、INFO、DEBUG日志输出格式
  appenders:
    - type: file
      threshold: ERROR
      currentLogFilename: ./logs/err.log
      archivedLogFilenamePattern: ./logs/err-%d.log.gz
      archivedFileCount: 7

    - type: file
      threshold: INFO
      currentLogFilename: ./logs/info.log
      archivedLogFilenamePattern: ./logs/info-%d.log.gz
      archivedFileCount: 7
      
    - type: file
      threshold: DEBUG
      currentLogFilename: ./logs/debug.log
      archivedLogFilenamePattern: ./logs/debug-%d.log.gz
      archivedFileCount: 7

  loggers:
    # 为此包内的所有类单独设置日志格式
    "com.chance.data.receiver.service.resources":
      level: INFO
      # 不再将这些类的日志输出到默认日志文件中
      additive: false
      # 指定新的日志输出格式
      appenders:
        # 输出到文件中
        - type: file
          # 文件路径及文件名
          currentLogFilename: ./logs/example.log
          # 文件压缩的格式，此处为按天压缩成gz的格式
          archivedLogFilenamePattern: ./logs/example-%d{yyyy-MM-dd}.log.gz
          # log输出格式，此处只输出代码中传入的原始log内容
          logFormat: "%msg"
          # 保留的日志文件数目，包括gz压缩文件数目，此处设置为日志文件保存近15天的数据
          archivedFileCount: 15

# 接口服务相关设置
server:
  applicationConnectors:
    # 服务类型为http
    - type: http
      # 服务端口
      port: 34348
  adminConnectors:
    - type: http
      port: 34349
```
使用maven将完成的dropwizard工程进行打包，对应的maven pom文件配置参照dropwizard [Getting Started][dropwizard getting started]页面。使用`mvn install`或是使用eclipse等ide将工程打成可执行jar包。

将jar包上传到EC2对应的接口目录下，使用命令`java -jar ****.jar server ****.yaml`即可启动服务，其中server后面传入的yaml文件名即为上方代码块中的配置文件。






[1]: http://docs.aws.amazon.com/zh_cn/lambda/latest/dg/welcome.html
[2]: http://docs.aws.amazon.com/zh_cn/lambda/latest/dg/setting-up.html
[3]: http://docs.aws.amazon.com/zh_cn/lambda/latest/dg/setup-awscli.html
[4]: http://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-getting-started.html
[5]: http://docs.aws.amazon.com/zh_cn/general/latest/gr/rande.html
[6]: http://docs.aws.amazon.com/zh_cn/lambda/latest/dg/getting-started-create-function.html
[mParticle wiki]: https://github.com/mParticle/mparticle-sdk-java/wiki
[mParticle firehose sample]: https://github.com/mParticle/firehose-sample
[mParticle sample build.gradle]: https://github.com/mParticle/firehose-sample/blob/master/sample-extension/build.gradle
[mParticle java api]: http://docs.mparticle.com/includes/java-sdk-javadocs/com/mparticle/sdk/model/eventprocessing/package-summary.html
[mParticle Module Registration]: https://github.com/mParticle/mparticle-sdk-java/wiki/Module-Registration#1-creating-your-moduleregistrationresponse-and-describing-your-service
[mParticle Event Processing]: https://github.com/mParticle/mparticle-sdk-java/wiki/Event-Processing 
[mParticle EventProcessingRequest.json]: https://github.com/mParticle/mparticle-sdk-java/blob/master/mparticle-sdk-java-samples/json/EventProcessingRequest.json
[aws s3]: http://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/Welcome.html
[aws s3client java api]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/
[okhttp home page]: http://square.github.io/okhttp/
[aws lambda envrionment variables]: http://docs.aws.amazon.com/zh_cn/lambda/latest/dg/env_variables.html
[mParticle Deployment]: https://github.com/mParticle/mparticle-sdk-java/wiki/Deployment
[aws ec2]: https://aws.amazon.com/cn/documentation/ec2/?icmpid=docs_menu
[dropwizard document]: http://www.dropwizard.io/0.9.1/docs/getting-started.html
[dropwizard getting started]: http://www.dropwizard.io/0.9.1/docs/getting-started.html














