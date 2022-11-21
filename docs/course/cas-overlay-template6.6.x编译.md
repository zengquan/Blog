# CAS server 6.6.x 编译

---------

## 下载[cas-overlayy-template6.6.x](https://github.com/apereo/cas-overlay-template.git)代码


## 证书生成

[使用 KeyStore Explorer (一个 keytool 的 GUI 工具) 签发带 SAN 的二级证书并导出为 p12 格式在 SpringBoot 中使用](https://blog.csdn.net/halozhy/article/details/121888033)

```
1.新建keyStore，在/src/main/resource/keystore/cas.p12

2.创建根证书设置密码
Common Name（CN）：公用名称，对于SSL证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端证书则为证书申请者的姓名；
Organization Unit（OU）：其他内容
Organization Name（O）：单位名称，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端单位证书则为证书申请者所在单位名称
Locality Name（L）：所在城市
State Name（ST）：所在省份，也有用“Provice”
Country（C）：所在国家，例如 中国 => CN
```
## 构建与运行

1. 要求JDK11、下载当前CAS版本为6.6

2. 修改tasks.gradle文件中的`createKeystore`任务，相关参数修改为证书的实际内容

3. 或者为tomcat配置证书

4. 在`cas-overlay-template`目录执行命令`gradlew.bat clean build`，会生成build目录

5. 解压`build/overlays/bootWar/cas/WEB-INF/lib/cas-server-webapp-resources-6.6.0.jar`文件，复制该文件下的内容到`src/main/resources/`目录下

6. 修改`src/main/resources/application.properties`中的下列参数

```text
server.ssl.key-store=KeyStore的文件路径
server.ssl.key-store-password=KeyStore的密码
server.ssl.key-password= 证书密码
server.ssl.enabled=true
```

7. 通过命令`gradlew.bat run -Pvalidate=false`来启动（或者在`gradle.properties`中添加参数`validate=false`）

8. 访问`https://localhost:8443/cas/login` 输入默认账号/密码`actuator/Mellon`进行登录

9. 访问`https://localhost:8443/cas/logout` 进行登出
