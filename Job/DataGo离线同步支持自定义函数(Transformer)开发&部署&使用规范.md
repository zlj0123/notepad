## 1 开发规范
* DataGo离线同步支持自定义函数，本质上是集成了DataX的Transformer功能，如果用户已经了解了DataX的Transformer功能和开发，可以跳过此开发规范。
* 本开发规范主要是用demo的形式来讲解如何去开发一个自定义函数，用户可以通过拷贝和修改demo，去实现自己的自定义函数。
* 一个自定义函数只实现一个功能，不要在一个自定义函数里实现很多功能，然后通过参数配置去选择使用哪个自定义函数，这不是一个好的开发方式。

demo很简单，把你配置的某个字段的值，替换成你配置的值，我们会标出一些步骤和需要注意的点，然后会附上整个demo。
### 1.1 pom.xml文件定义
* 必须要引入datax-common和datax-trnsformer依赖，demo里我们直接放到工程的libs里了，让你拿来就可以运行和上手。实际开发里你可以放在maven仓库里，然后引入，但是scope为provided，不需要打包进最后的发布包。
* 需要引入slf4j-api、logback-classic依赖，datax里用这个作为log输入记录。
* 其他你自己实现自定义函数用到的依赖，比如加解密工具。
```
<dependencies>
    <dependency>
	    <groupId>com.alibaba.datax</groupId>
        <artifactId>datax-common</artifactId>
        <version>${datax-version}</version>
        <scope>system</scope>
        <systemPath>${basedir}/src/main/libs/datax-common-3.0.0-hundsun.jar</systemPath>
        <exclusions>
	        <exclusion>
		        <artifactId>slf4j-log4j12</artifactId>
                <groupId>org.slf4j</groupId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
	    <groupId>com.alibaba.datax</groupId>
        <artifactId>datax-transformer</artifactId>
        <version>${datax-version}</version>
        <scope>system</scope>
        <systemPath>${basedir}/src/main/libs/datax-transformer-3.0.0-hundsun.jar</systemPath>
    </dependency>
    <dependency>
	    <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j-api-version}</version>
    </dependency>
    <dependency>
	    <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>${logback-classic-version}</version>
    </dependency>
</dependencies>
```

* build->plugins里需要引入maven-assembly-plugin，主要的集成内容在src/main/assembly/package.xml里。
```
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
	    <descriptors>
		    <descriptor>src/main/assembly/package.xml</descriptor>
        </descriptors>
        <finalName>datax</finalName>
    </configuration>
    <executions>
	    <execution>
		    <id>dwzip</id>
            <phase>package</phase>
            <goals>
	            <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 1.2 package.xml文件定义
* 作用就是打包，定义哪些文件、哪些依赖需要打包到最终的Transformer插件里去，以及插件的目录结构。
* 内容参照demo里的package.xml。
* 注意事项：1. 插件命名(打包出的目录)必须统一且唯一，比如这里叫```hundsun_demo```，这个命名跟代码setTransformerName里的命名、transformer.sjon里的name、上游使用自定义函数的命名保持一致。2. 命名不能以dx_开头，这个是DataX内部插件的命名开头。3. 必须把src/main/resources/transformer.json文件放到最终包的目录下，这个文件也有自己的要素。

### 1.3 resource/transformer.sjon文件定义
定义name和class，name跟上面的名字一致；class就是下面的自定义函数的具体实现类，DataX根据你配置的transformer名字，去找对应的目录名字->目录下的transformer.json文件->class进行注册、加载和运行。
```
{  
    "name": "hundsun_demo",  
    "class": "com.hundsun.rdc.bdata.datago.transformer.TransformerDemo",  
    "description": "transformer demo",  
    "developer": "ZhangLijun"  
}
```

### 1.4 自定义函数实现
实现参照demo。
datax-transformer里有两个抽象类，ComplexTransformer和Transformer类，我们根据需要实现这两个类的evaluate方法即可。
```
public abstract Record evaluate(Record var1, Map<String, Object> var2, Object... var3);
```
DataX的Reader读插件把数据读出来后，每条数据都会被抽象成一个Record对象，Record里有很多column（字段），你可以把Record看成是一个List，我们就是去处理这些column字段，把我们写的自定义函数运用到这些字段上。
比如下面的配置，会把你record里索引为11的字段取出来（索引从0开始），然后把这个字段的值运行一下你写的hundsun_demo插件的evaluate方法，这里其实就是把这个字段替换成"hundsun transformer demo"。
配置的时候必须配置name和parameter里的columnIndex这两个key，paras根据需要配置。

```
"transformer": [
                    {
                        "name": "hundsun_demo",
                        "parameter":
                            {
                            "columnIndex":11,
                            "paras":["hundsun transformer demo"]
                            }
                    }
                ]
```

evaluate方法里需要注意，取配置的columnIndex的值，永远是objects[0]，DataX框架里会把这个columnIndex的值放到object的第一个字段里，paras往后移动。
接下去就是取Record的字段，比如record.getColumn(columnIndex) -> **处理** -> record.setColumn(columnIndex，XxxColumn(value));
上面的处理就是你们要去实现的自定义函数的核心内容了，比如加解密，比如替换乱七八糟的字符等等；
XxxColumn是DataX内部定义的抽象Column，根据需要选择相应的Column。
![[Pasted image 20241014140259.png]]

### 1.5 打包结果
![[Pasted image 20241014141211.png]]

### 1.6 demo完整源码
用户完全可以基于这个源码编写自己的自定义函数。
下载地址：https://iknow.hs.net/portal/docView/home/120534

![[datax-transformer-demo.zip]]

## 2 部署规范
部署这里分两部分：Transformer插件部署和DataGo部署。

### 2.1 Transformer插件部署
这里假设你开发了一堆的Transformer插件，假设叫hundsun_demo1、hundsun_demo2、...... hundsun_demo10，然后你把这些插件都放置在linux目录hundsun_transformer下，结构如下：
```
/xxx/xxx/.../hundsun_transformer/
--------------------------------hundsun_demo1
--------------------------------hundsun_demo2
--------------------------------hundsun_demox
--------------------------------hundsun_demo10
```

备注：这里不管是你手动把这一堆Transformer插件拷贝到hundsun_transformer下，还是你自己制作一个see部署包，通过see部署到hundsun_transformer下，这个由你们自己决定。

### 2.2 DataGo部署
DataGo层都是通过see部署，然后有个see配置项，要求输入Transformer插件部署目录的绝对路径，上面就是/xxx/xxx/.../hundsun_transformer；当然，你没有transformer插件，你就空着，你的任务里也不能配置Transformer。
然后DataGo部署的时候，会建立一个链接，链接到这个目录下面
ln -s /xxx/xxx/.../hundsun_transformer /DataGo安装路径/bdata-datago/DataX/local_storage/transformer

我们通过建立软链接的方式调用你们的Transformer插件。

## 3 使用规范

### 3.1 DataX任务穿透


### 3.2 DataGo组件


### 3.3 大数据开发平台

