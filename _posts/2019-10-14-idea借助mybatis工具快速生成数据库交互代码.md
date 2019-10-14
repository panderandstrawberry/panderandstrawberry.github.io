## 问题：
之前有提到，建议大家使用mybatis逆向工程插件生成项目中数据库交互的代码，使用过程中发现有几个问题使用不是很方便：
1.逆向工程配置文件不会很好配置，新接触的人需要一段时间来学习配置项内容，XML的形式也不是很好理解。
2.针对微服务的形式，会存在多个项目都会有数据库交互，这样每个人项目都需要生成数据库的代码，存在重复的工作。
3.生成的Mapper代码，使用的时候不是很方便，在业务代码中，会存在写大量的重复的查询条件语句。
针对遇上遇到的问题，这里介绍使用IDEA的插件，来解决如上的问题。


### 解决问题1：
1.idea安装插件MybatisCodeHelper（现在插件已经变成收费的了，可以网上加下破解版）
2.ideade DataBase关联数据库
在如图所示的位置：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012150753144.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzMxMzk2NzY5,size_16,color_FFFFFF,t_70)
点击之后关联上后台的数据库，关联方式很简单，这里不做介绍，尝试加就可以了。
3.如图所示，使用插件生成所有表的交互接口：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012150936561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzMxMzk2NzY5,size_16,color_FFFFFF,t_70)
弹框如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012170745335.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzMxMzk2NzY5,size_16,color_FFFFFF,t_70)
这里可以根据正则表达式，来选择需要生成的表，对于库内存在很多不需要的表的时候很有效果，各个配置详细如图，都可以看得出来

可以使用正则表达是选择多个表生成代码
如：想生成yh表  则使用正则表达式  
```java
yh_t\w{1,}  
```
选择所有的表.
生成的代码目录结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191014095044354.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzMxMzk2NzY5,size_16,color_FFFFFF,t_70)

### 解决问题2：
多个微服务都会使用数据库交互操作时，将数据库交互的操作单独封装出一个包，所有的微服务都使用这个包，就可以有数据库的交互操作。
框架层新建 table-operation 项目，此项目只做基本的数据库交互操作，可提供远程调用的接口（优先度低），其他微服务使用此包的数据库操作接口。

### 解决问题3：
生成代码除了生成Mapper接口之外，还自动生成基本业务接口跟实现类代码，如图所示：
自动生成业务接口：
```java
package com.iasp.tableoperation.singletable.service;

public interface TbankinsstockService{


}

```
自动生成业务接口的实现类
```java
package com.iasp.tableoperation.singletable.serviceimpl;

import org.springframework.stereotype.Service;
import javax.annotation.Resource;
import com.iasp.tableoperation.singletable.mapper.TbankinsstockMapper;
import com.iasp.tableoperation.singletable.service.TbankinsstockService;
@Service
public class TbankinsstockServiceImpl implements TbankinsstockService{

    @Resource
    private TbankinsstockMapper tbankinsstockMapper;

}

```
接口和接口的实现类都自动生成，省掉新建文件带来的工作量，同时，基础业务接口的不断封装，在多个微服务项目开发使用时，公共的部分，代码开发的工作量都可以节省下来（此部分功能有点类似DB中的将AS独立封装出来的效果）


使用插件的缺陷：
1.生成java文件的名字有局限性，全部生成的话，名字不支持自定义，根据数据库的表明生成实体类接口的名字，但是可以简单地去掉前缀。
2.生成的数据类型有个别有问题：
如：业务使用的是double，但是实体类会生成BigDecimal
此问题插件没找大解决的办法，第一次全部生成的时候可以全局替换，之后单个生成的时候，手动改下即可，也可以通过基类封装转换方法解决，或者遇到之后，手动修改掉，只需要改一次就可以，再或者，业务直接使用BigDecimal类型，其实计算起来会更加准确，避免double计算精度失真的问题。
优点：
1.不需要反复创建实体类，接口，一次创建，永久只用。
2.单表不需要写SQL，避免SQL书写的错误
3.公共的接口只写一遍，简化开发，避免出错。
4.方便维护，数据库变更，只需要更新实体类即可。
5.可视化工具，方便操作。

包体大小：
yh  pub  zl全部表  bean+Mapper接口+业务接口+ 实现类  供476张表，打成jar包之后，差不多  42M左右。
