---
layout: post
title: "MySQL适配Geometry类型"
date: 2025-03-25 
description: "MySQL,MyBatisPlus,Geometry"

tag: 博客
---
- ## 概述

  **MySql**本身支持地理空间几何数据类型，但是Java里面没有对应的数据类型匹配，当查询该字段信息时，**Mysql**直接返回blob类型的数据，也就是字节流数据，基于以上原因，通过Mybatis本身良好的扩展性对**Geometry**类型做了简单适配。
- ## 适配环境

  1. MySql 8.18
  2. SpringBoot 2.1.+
  3. MybatisPlus 3.2.0
  4. vividsolutions jts 1.13  空间数据操作类库
- ## 适配过程
  1. 自定义Typehandler，继承Mybatis框架的**TypeHandler**类，重写**get**类的方法，handler的作用是当从MySql中查询数据的时候，自动将Geometry类型的数据转换成Java类型，目前适配了两种类型，**Point**和**LineString**，示例代码如下：

     ```java
     @MappedTypes(value = {List.class})
     public class MysqlGeoLineTypeHandler extends BaseTypeHandler<List<GeoPoint>> {

         private int srid = 0;
         private WKBReader wkbReader;
         private WKBWriter wkbWriter;

         public MysqlGeoLineTypeHandler() {
             GeometryFactory geometryFactory = new GeometryFactory(new PrecisionModel(), srid);
             wkbReader = new WKBReader(geometryFactory);
             this.wkbWriter = new WKBWriter();
         }

         public MysqlGeoLineTypeHandler(int srid) {
             this.srid = srid;
             GeometryFactory geometryFactory = new GeometryFactory(new PrecisionModel(), srid);
             wkbReader = new WKBReader(geometryFactory);
         }

         @Override
         public void setNonNullParameter(PreparedStatement ps, int i, List<GeoPoint> parameter, JdbcType jdbcType) throws SQLException {
         }

         @Override
         public List<GeoPoint> getNullableResult(ResultSet rs, String columnName) throws SQLException {
             return fromMysqlWkb(rs.getBytes(columnName));
         }

         @Override
         public List<GeoPoint> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
             return fromMysqlWkb(rs.getBytes(columnIndex));
         }

         @Override
         public List<GeoPoint> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
             return fromMysqlWkb(cs.getBytes(columnIndex));
         }

         private List<GeoPoint> fromMysqlWkb(byte[] bytes) {
             if (bytes == null) {
                 return null;
             }
             try {
                 byte[] geomBytes = ByteBuffer.allocate(bytes.length - 4).order(ByteOrder.LITTLE_ENDIAN)
                         .put(bytes, 4, bytes.length - 4).array();
                 Geometry geometry = wkbReader.read(geomBytes);
                 return Arrays.stream(geometry.getCoordinates())
                         .map(p -> new GeoPoint(new BigDecimal(String.valueOf(p.x)), new BigDecimal(String.valueOf(p.y))))
                         .collect(Collectors.toList());
             } catch (Exception ignored) {
             }
             return null;
         }
     }

     ```

     以上代码是用来转换**LineString**类型的**Geometry**数据，**Point**类型数据适配类似以上代码，代码中使用了**jts**库提供的数据转换API，负责将**blob**类型的字节流数据转换为**Geometry**类型的**Model**。

  2. 自定义**insert**和**update**方法，用来进行**Geometry**类型数据的插入和修改，目前采用的是编写**sql**方式进行数据插入，也可以编写**Provider**，通过Java代码动态生成sql，示例代码如下：

     ```sql
         <insert id="insertLineGeo">
            INSERT INTO line
        <trim prefix="(" suffix=")" suffixOverrides=",">
            <if test="id != null">id,</if>
            <if test="name != null">name,</if>
            <if test="devAlarm != null">dev_alarm,</if>
            <if test="speedLimit != null">speed_limit,</if>
            <if test="remark != null">remark,</if>
            <if test="createTime != null">create_time,</if>
            <if test="updateTime != null">update_time,</if>
            <if test="line != null">line,</if>
        </trim>
        <trim prefix="VALUES(" suffix=")" suffixOverrides=",">
            <if test="id != null">#{id},</if>
            <if test="name != null">#{name},</if>
            <if test="devAlarm != null">#{devAlarm},</if>
            <if test="speedLimit != null">#{speedLimit},</if>
            <if test="remark != null">#{remark},</if>
            <if test="createTime != null">#{createTime},</if>
            <if test="updateTime != null">#{updateTime},</if>
            <if test="line != null and line.size() > 1">
                <foreach item="item" index="index" collection="line" open="ST_GeomFromText('LINESTRING(" separator="," close=")')">${item.longitude} ${item.latitude}</foreach>
            </if>
        </trim>
      </insert>
     ```

     以上代码用来向数据库中插入**LineString**类型的数据，**update**方法类似，主要关注的地方是将MySql自带的转换函数加入到**MyBatis**的**sql mapper**中，使**MyBatis**可以生成适配**Geometry**类型的sql语句。**MySql**关于**Geometry**类型数据处理的相关函数可参看[官方文档](https://dev.mysql.com/doc/refman/8.0/en/spatial-function-reference.html))，可以根据自己的需求适配不同的数据类型，**MySql**支持的[数据类型](https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html)见文档。
3. 配置typeHandler 
   1. 本例子基于**SpringBoot**构建，在**application.yml**中配置自定义handler，配置片段如下：

      ```yml
      # mybatis-plus配置
      mybatis-plus:
      # mapper对应文件
      mapper-locations: classpath:mapper/*.xml
      # 实体扫描，多个package用逗号或者分号分隔
      type-aliases-package: com.sanygroup.pdds.system.model
      configuration:
       log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
      # 配置自定义typehandler
      type-handlers-package: com.sanygroup.pdds.map.type
      ```
   2. Geo类型**Model**中添加注解，代码如下：

      ```java
      @ApiModelProperty(value = "路线")
      @TableField(typeHandler = MysqlGeoLineTypeHandler.class)
      private List<GeoPoint> line;
      ```
   3. **Model Mapper**中添加**TypeHandler**，代码如下：

      ```xml
      <!-- 通用查询映射结果 -->
      <resultMap id="BaseResultMap" type="com.sanygroup.pdds.map.model.Line">
          <id column="id" property="id"/>
          <result column="name" property="name"/>
          <result column="dev_alarm" property="devAlarm"/>
          <result column="speed_limit" property="speedLimit"/>
          <result column="remark" property="remark"/>
          <result column="line" property="line" typeHandler="com.sanygroup.pdds.map.type.MysqlGeoLineTypeHandler"/>
      </resultMap>
      ```

- 总结

  1. MySql内置的空间几何数据类型可以减少表的复杂度，同时可以进行复杂的位置查询操作。
  2. 以上示例只是一些简单应用，高级用法可以参看[官方文档](https://dev.mysql.com/doc/refman/8.0/en/)。

     ![jaHWpVwILDgN6Xb](https://i.loli.net/2019/12/11/jaHWpVwILDgN6Xb.gif)
