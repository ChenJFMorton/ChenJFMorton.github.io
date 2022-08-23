---
title: spring如何在普通类里面进行依赖注入
date: 2021-11-08 15:14:09
hidden: true
tags:
---

功能需求:
因项目需要,要做一个导入导出功能,以此来生成批量数据.为了让导入功能和业务逻辑分开,就把这个导入功能做成一个独立的Util工具类.

问题描述
把导入功能和业务逻辑分开后，所遇到的问题就是,这个导入功能需要依赖其他的资源,按照一般的注解方法肯定是行不通了,那么，如何在普通方法里面进行注解依赖呢？

猜想
一般的spring注解(@controller 、@service、@repository等等)这些注解的作用就是把这些类纳入进spring容器中进行管理。如果我们想要在普通类里面进行资源的依赖注入，第一步就先要实现该类能被spring容器管理。如何实现呢？

解决方案
1、注册方法
    在类名上方加入 @Component 注解(和普通的控制器注解类似)
```
//注册扫描普通类
@Component
public class ParsingUploadRooms {
}
```
2、注入需要使用的资源
```
    @Resource
    private RoomTypeManager roomTypeManager;

    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    //需要调用本身,因此也需要实例化
 	private static ParsingUploadRooms uploadRooms;
```
3、初始化注入(凡是需要依赖注入的资源,都要在下面初始化,不然诸如不成功别忘了 @PostConstruct 注解)
```
    @PostConstruct
    public void init() {
        uploadRooms = this;
        uploadRooms.jdbcTemplate = this.jdbcTemplate;
        uploadRooms.roomTypeManager = this.roomTypeManager;
     
    }
```
4、然后就可以像平时使用的spring组件一样这个工具类了.
5、完整实例
```
       
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataAccessException;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.annotation.Resource;
import java.util.Map;

/**
 * 〈一句话功能简述〉<br>
 * 〈解析上传的文件里面的数据，存入数据库，实现导入式新建房源〉
 *
 * @author wxz
 * @create 2019-04-09
 * @since 1.0.0
 */
//注册扫描普通类
@Component
public class ParsingUploadRooms {
    
    @Resource
    private RoomTypeManager roomTypeManager;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    private static ParsingUploadRooms uploadRooms;

    private IdList idList = new IdList();

    @PostConstruct
    public void init() {
        uploadRooms = this;
        uploadRooms.jdbcTemplate = this.jdbcTemplate;

        uploadRooms.roomTypeManager = this.roomTypeManager;
    }

    public IdList getIdList(){
        IdList idList = ParsingUploadRooms.uploadRooms.idList;
        uploadRooms.jdbcTemplate.XX();
        return idList;
    }
}
```

https://blog.csdn.net/qq_37844454/article/details/89230199