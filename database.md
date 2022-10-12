# 城信图语数据库部署文档
[toc]
## 1. 关于本文档

本文档旨在厘清城信图语的数据表与功能模块之间的关系，便于将城信图语按需进行功能模块裁剪部署。随着城信图语的不断功能迭代开发，本文档也会不断同步更新维护。



## 2. 数据库准备

城信图语的数据库要求是PostgreSQL ＋ PostGIS。

### 2.1 Docker安装

首先拉取PostgreSQL和PostGIS的镜像，我们推荐使用kartoza/postgis镜像，此镜像把PostgreSQL和PostGIS镜像集合在一起了，方便使用。

```bash
docker pull kartoza/postgis:13.0
```

在Docker镜像拉取后，接着我们创建Docker容器。

```bash
docker run --name postgres -p 5432:5432 -e POSTGRES_USER=onemap -e POSTGRES_PASSWORD=onemap -e POSTGRES_DBNAME=onemap -d kartoza/postgis:13.0
```

至此，我们的数据库已安装完毕。

### 2.2 数据库初始化

首先我们要启用PostGIS扩展，在用SQL客户端工具（psql、Navicat、pgAdmin等）登录数据库后，执行下面SQL命令。

```sql
CREATE EXTENSION postgis;
```

我们还需要创建城信图语的数据库架构，执行下面SQL命令。

```sql
CREATE SCHEMA onemap;
```
至此，数据库的环境已准备完毕。



## 3. 必要的基础数据库表

以下数据库表是城信图语后台启动的必要数据库表。



### 3.1 访问信息表 (access_info)

access_info表用于记录各API接口的访问统计，是必要的基础数据库表，一定需要创建，创建的SQL语句如下：

```sql
--创建access_info表ID序列
CREATE SEQUENCE IF NOT EXISTS access_info_id_seq;
--创建access_info表
CREATE TABLE IF NOT EXISTS access_info (
  id int4 NOT NULL DEFAULT nextval('access_info_id_seq'::regclass),
  classname varchar,
  methodname varchar,
  displayname varchar,
  apiurl varchar,
  apimethod varchar,
  count int8,
  createtime timestamptz(6),
  lastatime timestamptz(6)
);
```



### 3.2 用户信息表 (users)

users表用户记录用户信息，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建users用户信息表
CREATE TABLE IF NOT EXISTS users (
  id int8 NOT NULL PRIMARY KEY,
  iconid int8,
  name varchar(255),
  displayname varchar(255),
  password varchar(255)
);
```



### 3.3 标签内容表 (tag、tag_content)

tag_content表用于记录标签，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建tag表ID序列
CREATE SEQUENCE IF NOT EXISTS tag_id_seq;
--创建tag表
CREATE TABLE IF NOT EXISTS tag (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('tag_id_seq'::regclass),
  name varchar(255),
  count int4,
  type varchar(32) NOT NULL
);
--创建tag_content表ID序列
CREATE SEQUENCE IF NOT EXISTS tag_content_id_seq;
--创建tag_content表
CREATE TABLE IF NOT EXISTS tag_content (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('tag_content_id_seq'::regclass),
  tagid int8,
  content varchar(255),
  contentid int8,
  name varchar(255)
);
```



### 3.4 用户权限表 (userrights)

userrights表用于设置用户的各模块内容的权限设置，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建userrights表ID序列
CREATE SEQUENCE IF NOT EXISTS userrights_id_seq;
--创建userrights表
CREATE TABLE IF NOT EXISTS userrights (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('userrights_id_seq'::regclass),
  userorroleid int8 NOT NULL,
  resourcetype character varying(255),
  resourceid int8,
  operations int8 DEFAULT 0,
  forbidden int8 DEFAULT 0
);
--设置唯一约束
ALTER TABLE userrights ADD CONSTRAINT userrights_unique_cons UNIQUE (userorroleid, resourcetype, resourceid);
```




### 3.5 用户偏好表 (userpreference)

userpreference表用于记录用户偏好，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建userpreference表ID序列
CREATE SEQUENCE IF NOT EXISTS userpreference_id_seq;
--创建userpreference表
CREATE TABLE IF NOT EXISTS userpreference (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('userpreference_id_seq'::regclass),
  userid int8,
  content varchar(255),
  description varchar(255),
  weight int4,
  createtime timestamp(6),
  preferencetype int2,
  contenttype varchar(255)
);
```



### 3.6 类型定义表 (type)

type表用于记录类型的定义表，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建type表
CREATE TABLE IF NOT EXISTS type (
  id int8 NOT NULL PRIMARY KEY,
  pid int8,
  name varchar(255),
  type int4 DEFAULT 0,
  displayname varchar(255)
);
```



### 3.7 资源内容类型表 (contenttype)

contenttype表用于记录系统本schema下的内容资源类型，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建资源内容类型表contenttype
CREATE TABLE IF NOT EXISTS contenttype (
	name varchar(255) NOT NULL PRIMARY KEY,
	tablename varchar(255),
	idfield varchar(255),
	filtercondition varchar(255),
	displayname varchar(255),
	description varchar(255),
	sindex int4,
	canlike bool DEFAULT false,
	cancollect bool DEFAULT false,
	canfollow bool DEFAULT false,
	canshare bool DEFAULT false,
	cancomment bool DEFAULT false
);
-- ----------------------------
-- Records of contenttype
-- ----------------------------
INSERT INTO contenttype VALUES ('mapsign', 'mapsign', 'id', NULL, '地图标记', '城信图语地图标记资源', NULL, 't', 't', 't', 't', 't');
INSERT INTO contenttype VALUES ('topicmeta', 'topicmeta', 'id', NULL, '专题', '城信图语专题资源', NULL, 't', 't', 't', 't', 't');
INSERT INTO contenttype VALUES ('filemeta', 'filemeta', 'id', NULL, '文件', '城信图语文件资源', NULL, 't', 't', 't', 't', 't');
INSERT INTO contenttype VALUES ('datacatalog', 'datacatalog', 'id', NULL, '数据资源目录', '城信图语数据资源目录', 0, 't', 't', 't', 't', 't');
```




### 3.8 通用数据字典表 (dict)

dict表用于记录通用的数据字典，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建dict表
CREATE TABLE IF NOT EXISTS dict (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(255),
  displayname varchar(255),
  content varchar(255),
  isdefault bool,
  weight int4,
  pid int8
);
-- ----------------------------
-- Records of dict
-- ----------------------------
INSERT INTO dict VALUES (16, 'registertime_desc', '注册时间最新', 'ServerSearchOrder', 't', 6, NULL);
INSERT INTO dict VALUES (17, 'registertime_asc', '注册时间最久', 'ServerSearchOrder', 'f', 5, NULL);
INSERT INTO dict VALUES (18, 'displayname_asc', '名称排序', 'ServerSearchOrder', 'f', 4, NULL);
INSERT INTO dict VALUES (42, '2', 'ArcGISMapserver(无切片方案)', 'RegisterServerType', 'f', 3, NULL);
INSERT INTO dict VALUES (40, '0', '数据表', 'RegisterServerType', 'f', 5, NULL);
INSERT INTO dict VALUES (41, '1', 'ArcGISMapserver(包含切片方案)', 'RegisterServerType', 'f', 4, NULL);
INSERT INTO dict VALUES (43, '3', 'ArcGIS矢量切片', 'RegisterServerType', 'f', 2, NULL);
INSERT INTO dict VALUES (44, '4', 'GEOJSON', 'RegisterServerType', 't', 1, NULL);
INSERT INTO dict VALUES (109, '0', 'geojson', 'ServerDisplayType', 'f', 6, NULL);
INSERT INTO dict VALUES (110, '10', 'postgis矢量切片', 'ServerDisplayType', 'f', 6, NULL);
INSERT INTO dict VALUES (37, 'new', '创建时间最新', 'EchartGraphOrder', 'f', 3, NULL);
INSERT INTO dict VALUES (38, 'older', '创建时间最久', 'EchartGraphOrder', 'f', 2, NULL);
INSERT INTO dict VALUES (39, 'name', '创建名称', 'EchartGraphOrder', 'f', 1, NULL);
INSERT INTO dict VALUES (158, 'all', '全部', 'EchartGraphType', 'f', 4, NULL);
INSERT INTO dict VALUES (33, 'pie', '饼图', 'EchartGraphType', 'f', 3, NULL);
INSERT INTO dict VALUES (34, 'bar', '柱状图 ', 'EchartGraphType', 'f', 2, NULL);
INSERT INTO dict VALUES (35, 'line', '折线图', 'EchartGraphType', 'f', 1, NULL);
INSERT INTO dict VALUES (36, 'title', '标题', 'EchartGraphType', 'f', 1, NULL);
INSERT INTO dict VALUES (53, 'older', '创建时间最久', 'PortraitManagerOrder', 'f', 1, NULL);
INSERT INTO dict VALUES (52, 'new', '创建时间最新', 'PortraitManagerOrder', 'f', 2, NULL);
INSERT INTO dict VALUES (150, '2', '标准上限', 'IndexManagerMoreData', 'f', 6, NULL);
INSERT INTO dict VALUES (151, '3', '数据颗粒', 'IndexManagerMoreData', 'f', 5, NULL);
INSERT INTO dict VALUES (152, '4', '目标', 'IndexManagerMoreData', 'f', 4, NULL);
INSERT INTO dict VALUES (153, '5', '数据填报部门', 'IndexManagerMoreData', 'f', 3, NULL);
INSERT INTO dict VALUES (154, '6', '数据统计联系人', 'IndexManagerMoreData', 'f', 2, NULL);
INSERT INTO dict VALUES (155, '7', '数据统计联系人电话', 'IndexManagerMoreData', 'f', 1, NULL);
INSERT INTO dict VALUES (149, '1', '标准下限', 'IndexManagerMoreData', 'f', 7, NULL);
INSERT INTO dict VALUES (6, 'str', '字符串', 'ServerAtrOptionType', 't', 9, NULL);
INSERT INTO dict VALUES (7, 'num', '数值', 'ServerAtrOptionType', 'f', 8, NULL);
INSERT INTO dict VALUES (8, 'time', '时间', 'ServerAtrOptionType', 'f', 7, NULL);
INSERT INTO dict VALUES (9, 'space', '空间', 'ServerAtrOptionType', 'f', 6, NULL);
INSERT INTO dict VALUES (54, 'alltype', '全部', 'PortraitModuleType', 'f', 5, NULL);
INSERT INTO dict VALUES (55, 'staticgraph', '统计图表', 'PortraitModuleType', 'f', 3, NULL);
INSERT INTO dict VALUES (56, 'title', '标题', 'PortraitModuleType', 'f', 3, NULL);
INSERT INTO dict VALUES (57, 'text', '文本', 'PortraitModuleType', 'f', 3, NULL);
INSERT INTO dict VALUES (58, 'table', '表格', 'PortraitModuleType', 'f', 2, NULL);
INSERT INTO dict VALUES (59, 'map', '地图', 'PortraitModuleType', 'f', 2, NULL);
INSERT INTO dict VALUES (60, 'linkcomb', '联动组合', 'PortraitModuleType', 'f', 1, NULL);
INSERT INTO dict VALUES (144, '0', '第三方地址', 'ServerSpecPageType', 't', 3, NULL);
INSERT INTO dict VALUES (145, '1', '内部地址', 'ServerSpecPageType', 'f', 2, NULL);
INSERT INTO dict VALUES (146, '2', '配置的专题', 'ServerSpecPageType', 'f', 1, NULL);
INSERT INTO dict VALUES (27, '2016', '2016年', 'ServerBaseOptionTime', 'f', 3, NULL);
INSERT INTO dict VALUES (24, '2019', '2019年', 'ServerBaseOptionTime', 't', 6, NULL);
INSERT INTO dict VALUES (25, '2018', '2018年', 'ServerBaseOptionTime', 'f', 5, NULL);
INSERT INTO dict VALUES (26, '2017', '2017年', 'ServerBaseOptionTime', 'f', 4, NULL);
INSERT INTO dict VALUES (28, '2015', '2015年', 'ServerBaseOptionTime', 'f', 2, NULL);
INSERT INTO dict VALUES (111, '', '无', 'ServerBaseOptionTime', 'f', 1, NULL);
INSERT INTO dict VALUES (114, '2020', '2020年', 'ServerBaseOptionTime', 'f', 7, NULL);
INSERT INTO dict VALUES (115, '2021', '2021年', 'ServerBaseOptionTime', 'f', 8, NULL);
INSERT INTO dict VALUES (157, '10', 'postgis矢量切片', 'mapStyleExpression', 'f', 1, NULL);
INSERT INTO dict VALUES (156, '0', 'GEOJSON', 'mapStyleExpression', 'f', 2, NULL);
INSERT INTO dict VALUES (159, '1', '层次分析法', 'IndexMethod', 'f', 3, NULL);
INSERT INTO dict VALUES (160, '2', '专家评价法', 'IndexMethod', 'f', 2, NULL);
INSERT INTO dict VALUES (161, '3', '客观评价法', 'IndexMethod', 'f', 1, NULL);
INSERT INTO dict VALUES (162, '4', '全部', 'IndexMethod', 'f', 4, NULL);
```



### 3.9 平台字典表 (platformmeta)

platformmeta表用于记录平台的数据字典，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建platformmeta表
CREATE TABLE IF NOT EXISTS platformmeta (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(64),
  description varchar(255),
  platformtype int8,
  baseurl varchar(255),
  srid int4
);
-- ----------------------------
-- Records of platformmeta
-- ----------------------------
BEGIN;
INSERT INTO platformmeta VALUES (1, 'default', '系统默认的平台', 1, NULL, NULL);
COMMIT;
```



### 3.10 中国行政区划 (china_all)

china_all表用于记录中国行政区划的数据，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建china_all表
CREATE TABLE IF NOT EXISTS china_all (
  id int8 NOT NULL PRIMARY KEY,
  pid int8,
  code varchar(20) NOT NULL,
  pcode varchar(20),
  name varchar(50),
  level varchar(20),
  telecode varchar(20),
  centerx float8 NOT NULL DEFAULT 0.0,
  centery float8 NOT NULL DEFAULT 0.0,
  sindex int4 NOT NULL DEFAULT 0,
  childrennum int4 NOT NULL DEFAULT 0,
  shape "public"."geometry",
  name_short varchar(255),
  centerlat float8,
  centerlng float8,
  iscapital bool,
  centerlat2 float8,
  centerlng2 float8
);
```

注意：此表的数据需要同步。


## 4. 功能模块数据库表清单


### 4.1 操作栏接口

#### 4.1.1 导航栏接口
需要***navigation***表，依赖***userroleright_resource***表。其中navigation表创建SQL如下：

```sql
--创建navigation表
CREATE TABLE IF NOT EXISTS navigation (
  id int8 NOT NULL PRIMARY KEY,
  icon varchar(255),
  icontype int4 DEFAULT 0,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  route varchar(255),
  frame int4,
  weight int4
);
-- ----------------------------
-- Records of navigation
-- ----------------------------
BEGIN;
INSERT INTO navigation VALUES (1, 'home', 0, NULL, '首页', NULL, '/layout', 1, 8);
INSERT INTO navigation VALUES (2, 'yzt', 0, NULL, '一张图', NULL, '/layout/onemap', 0, 7);
INSERT INTO navigation VALUES (3, 'ztfx', 0, NULL, '专题分析', NULL, '/layout/topic', 0, 6);
INSERT INTO navigation VALUES (4, 'zbjc', 0, NULL, '指标监测', NULL, '/layout/indexs', 0, 5);
INSERT INTO navigation VALUES (5, 'fxpj', 0, NULL, '体检评估', NULL, '/layout/evaluate', 0, 4);
INSERT INTO navigation VALUES (6, 'pgyj', 0, NULL, '评估预警', NULL, '/layout/warning', 0, 3);
INSERT INTO navigation VALUES (7, 'jcyj', 0, NULL, '监测预警', NULL, '/layout/monitor', 0, 2);
INSERT INTO navigation VALUES (8, 'pgbg', 0, NULL, '评估报告', NULL, '/layout/report', 0, 1);
COMMIT;
```

#### 4.1.2 工具栏接口

需要***toolbar***表，依赖***modelcatalog***表和***modelcatalog_id_seq***序列。其中toolbar表创建SQL如下：

```sql
--新建toolbar表的ID序列
CREATE SEQUENCE IF NOT EXISTS toolbar_id_seq;
--新建toolbar表
CREATE TABLE IF NOT EXISTS toolbar (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('toolbar_id_seq'::regclass),
  pid int8,
  icon varchar(255),
  icontype int4,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  weight int4,
  isdelete bool,
  modelid int8,
  createtime timestamp(6),
  creator int8,
  type int4 NOT NULL
);
-- ----------------------------
-- Records of toolbar
-- ----------------------------
BEGIN;
INSERT INTO toolbar VALUES (42, 33, 'kjtj', 0, 'caijianceshigongju', '裁剪测试工具', '裁剪模型分析测试工具', NULL, 'f', 27, '2021-06-07 14:21:36.158414', NULL, 1);
INSERT INTO toolbar VALUES (30, NULL, NULL, NULL, 'ditucaozuo', '地图操作', '「地图操作」工具栏分类', 4, 'f', NULL, NULL, NULL, 0);
INSERT INTO toolbar VALUES (31, NULL, NULL, NULL, 'xuanzecaozuo', '选择操作', '「选择操作」工具栏分类', 3, 'f', NULL, NULL, NULL, 0);
INSERT INTO toolbar VALUES (32, NULL, NULL, NULL, 'gaojigongju', '高级工具', '「高级工具」工具栏分类', 2, 'f', NULL, NULL, NULL, 0);
INSERT INTO toolbar VALUES (33, NULL, NULL, NULL, 'qita', '其他', '「其他」工具栏分类', 1, 'f', NULL, NULL, NULL, 0);
INSERT INTO toolbar VALUES (1, 30, 'reset', 0, 'full', '全局', NULL, 9999, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (5, 31, 'xz', 0, 'clickon', '点选', NULL, 9899, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (8, 31, 'measure', 0, 'measurement', '测量', NULL, 9895, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (11, 31, 'clear', 0, 'clear', '清除', NULL, 9896, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (17, 31, 'kjcx', 0, 'attribute', '属性', NULL, 9898, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (20, 31, 'tuli', 0, 'legend', '图例', NULL, 9897, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (12, 32, 'kjtj', 0, NULL, '空间体检', NULL, NULL, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (15, 32, 'kdfx', 0, NULL, '空间可达', NULL, NULL, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (13, 32, 'gjtj', 0, NULL, '高级统计', NULL, 3, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (14, 32, 'hmd', 0, 'kerneldensity', '核密度分析', NULL, 4, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (19, 33, 'ztt', 0, 'thematicmap', '专题图', NULL, 2, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (26, 33, 'kjtj', 0, 'huanchongquceshigongju', '缓冲区测试工具', '缓冲区分析测试工具按钮', NULL, 'f', 37, '2021-06-02 18:10:22.955488', 10007, 1);
INSERT INTO toolbar VALUES (3, 30, 'suoxiao', 0, 'zoomout', '缩小', NULL, 9997, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (89, 32, 'model-line', 0, 'kongguifenxi', '控规分析', NULL, NULL, 'f', 5, '2021-06-29 14:04:19.415961', 10044, 1);
INSERT INTO toolbar VALUES (2, 30, 'fangda', 0, 'zoomin', '放大', NULL, 9998, 'f', NULL, NULL, NULL, 1);
INSERT INTO toolbar VALUES (21, 32, 'tuli', 0, 'huanchongqufenxi', '缓冲区分析', '', 5, 'f', 37, NULL, NULL, 1);
INSERT INTO toolbar VALUES (16, 33, 'all', 0, 'splitscreen', '分屏', NULL, 1, 'f', NULL, NULL, NULL, 1);
COMMIT;
--修改序列的下一个值
ALTER SEQUENCE toolbar_id_seq RESTART WITH 90;
```



### 4.2 文件资源管理接口

文件资源管理接口需要***filemeta***数据表，它的创建SQL如下：

```sql
--创建filemeta表的ID序列
CREATE SEQUENCE IF NOT EXISTS filemeta_id_seq;
--创建filemeta表
CREATE TABLE IF NOT EXISTS filemeta (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('filemeta_id_seq'::regclass),
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  contentpath varchar(255),
  type varchar(255),
  originame varchar(255),
  creattime timestamp(6),
  creator int4,
  filesize int8,
  md5 varchar(255),
  suffix varchar(255),
  uid varchar(32),
  views int8,
  creattype int8,
  downloads int8,
  issys int4,
  extinfo varchar
);
```



### 4.3 地图数据资源管理接口

地图数据资源管理接口需要**文件资源管理接口**。

地图数据资源管理接口需要以下数据表：***layer***、***layer_field***、***layerstyles***、***mapserver***、***mapserver_layer***、***mapserver_monitor***、***cachejob***表，这些数据表的创建SQL如下：

```sql
--创建layer表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS layer_id_seq;
--创建layer表
CREATE TABLE IF NOT EXISTS layer (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('layer_id_seq'::regclass),
  layeridnum int4,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  spacetype int4,
  time int8,
  space int8,
  other int8,
  weight int4,
  style text,
  legend text,
  property varchar(255),
  pid int8,
  showproperty bool,
  nodetype int4,
  mapserverid int8,
  latestjobid int8,
  propertytype int2,
  srid int4,
  rectangle varchar(255),
  adcode int8,
  displaylevel int4[]
);
--创建layer_field表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS layer_field_id_seq;
--创建layer_field表
CREATE TABLE IF NOT EXISTS layer_field (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('layer_field_id_seq'::regclass),
  layerid int8,
  fieldname varchar(255),
  fielddisplayname varchar(255),
  fieldtype varchar(255),
  fielddescription varchar(255),
  isbou bool,
  showfield bool,
  length float8,
  precision int2,
  sourcetype varchar(255),
  weight int4,
  fieldsourcename varchar(255),
  propertytype int2
);
--创建layerstyles表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS layerstyles_id_seq;
--创建layerstyles表
CREATE TABLE IF NOT EXISTS layerstyles (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('layerstyles_id_seq'::regclass),
  name varchar(255),
  displayname varchar(255),
  layerid int4,
  datasourcetable varchar(255),
  displaytype int2,
  style varchar,
  moreinfo varchar(3000),
  weight int4,
  creator int8,
  creattime timestamp(6),
  updator int8,
  updatetime timestamp(6),
  legend varchar(3000),
  uid varchar,
  thumbnail int8,
  basemap varchar(255)
);
--创建mapserver表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS mapserver_id_seq;
--创建mapserver表
CREATE TABLE IF NOT EXISTS mapserver (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('mapserver_id_seq'::regclass),
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  type int4 NOT NULL,
  destination varchar(255),
  registertime timestamp(6),
  updatetime timestamp(6),
  registrant int4,
  isbasemap bool,
  origin varchar(255),
  thumbnail int8,
  agenttype int4,
  displaytype int4,
  moreinfo varchar,
  creattype int2,
  inheritedfrom int8
);
--创建mapserver_layer表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS mapserver_layer_id_seq;
--创建mapserver_layer表
CREATE TABLE IF NOT EXISTS mapserver_layer (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('mapserver_layer_id_seq'::regclass),
  mapserverid int8,
  name varchar(255),
  layerid int8
);
--创建mapserver_monitor表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS mapserver_monitor_id_seq;
--创建mapserver_monitor表
CREATE TABLE IF NOT EXISTS mapserver_monitor (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('mapserver_monitor_id_seq'::regclass),
  mapserverid int8,
  status int2,
  content varchar(255),
  time timestamp(6)
);
--创建cachejob表的ID字段序列
CREATE SEQUENCE IF NOT EXISTS cachejob_id_seq;
--创建cachejob表
CREATE TABLE IF NOT EXISTS cachejob (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('cachejob_id_seq'::regclass),
  layerid int8,
  starttime timestamp(6),
  status int2,
  progress float4,
  cacheuser varchar(255),
  stoptime timestamp(6),
  datasize float8,
  cachesize float8,
  reason text
);
```



### 4.4 数据目录接口

数据目录接口依赖**文件资源管理接口**和**地图数据资源管理接口**。

数据目录接口需要以下数据表：***datacatalog***、***datacatalog_detail***、***datacatalog_file***表，这些数据表的创建SQL如下：

```sql
--创建datacatalog表的ID序列
CREATE SEQUENCE IF NOT EXISTS datacatalog_id_seq;
--创建datacatalog表
CREATE TABLE IF NOT EXISTS datacatalog (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('datacatalog_id_seq'::regclass),
  pid int8,
  name varchar(255),
  description varchar(255),
  filtershow bool DEFAULT true,
  filterstyle text,
  nodetype int4,
  filterdata text,
  filtertype int2 DEFAULT 1,
  sindex int4,
  displayname varchar(255),
  creator int8,
  createtime timestamptz(0),
  updator int8,
  updatetime timestamptz(6),
  adcode varchar(255)
);
--创建datacatalog_detail表的ID序列
CREATE SEQUENCE IF NOT EXISTS datacatalog_detail_id_seq;
--创建datacatalog_detail表
CREATE TABLE IF NOT EXISTS datacatalog_detail (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('datacatalog_detail_id_seq'::regclass),
  datacatalogid int8,
  layerid int8 NOT NULL DEFAULT 0,
  isdefault bool,
  mapserverid int8 NOT NULL DEFAULT 0,
  time int8,
  space int8,
  other int8,
  weight int4,
  displaytype int2,
  layerstyleid int8 NOT NULL DEFAULT 0,
  displayname varchar(255)
);
--创建datacatalog_file表的ID序列
CREATE SEQUENCE IF NOT EXISTS datacatalog_file_id_seq;
--创建datacatalog_file表
CREATE TABLE IF NOT EXISTS datacatalog_file (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('datacatalog_file_id_seq'::regclass),
  pid int8,
  displayname varchar(255) COLLATE "pg_catalog"."default",
  datacatalogid int8,
  fileid int8 DEFAULT 0,
  sindex int4
);
```



### 4.5 专题配置接口

专题配置接口依赖**文件资源管理接口**。

专题配置接口需要改下数据表：***topicmeta***、***topic_detail***、***topictemplate***、***topictemplate_detail***、***topiccontenttemplate***表，这些数据表的创建SQL如下：

```sql
--创建topicmeta表
CREATE TABLE IF NOT EXISTS topicmeta (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(255),
  description varchar(255),
  templateid int4,
  url varchar(255),
  style varchar(2550),
  createtime timestamp(6),
  updatetime timestamp(6),
  creator int4,
  ispublic bool,
  thumbnail int8,
  copyby int8,
  displayname varchar(255),
  isdelete int2,
  updator int4,
  uid varchar,
  pageviews int8,
  platform int8,
  authmode int4 DEFAULT 0
);
--创建topic_detail表
CREATE TABLE IF NOT EXISTS topic_detail (
  id int8 NOT NULL PRIMARY KEY,
  topicid int8,
  pid int8,
  position text,
  name varchar(255),
  type varchar(255),
  content varchar(255),
  weight int4,
  modulestyle text,
  isdefault bool,
  displayname varchar(255)
);
--创建topictemplate表
CREATE TABLE IF NOT EXISTS topictemplate (
  id int8 NOT NULL PRIMARY KEY,
  pid int8,
  name varchar(255),
  displayname varchar(255),
  thumbnail varchar(255),
  style varchar(255)
);
-- ----------------------------
-- Records of topictemplate
-- ----------------------------
BEGIN;
INSERT INTO topictemplate VALUES (5, NULL, '网页版', '网页版', NULL, NULL);
INSERT INTO topictemplate VALUES (4, 5, '纯卡片', '纯卡片', '451', NULL);
INSERT INTO topictemplate VALUES (3, 5, '双列图文', '双列图文', '449', NULL);
INSERT INTO topictemplate VALUES (6, NULL, '手机版', '手机版', NULL, NULL);
INSERT INTO topictemplate VALUES (2, 7, '上下图文', '上下图文', '452', NULL);
INSERT INTO topictemplate VALUES (1, 7, '左右图文', '左右图文', '450', NULL);
INSERT INTO topictemplate VALUES (7, 5, '底图加卡片', '左图右文', '450', NULL);
COMMIT;
--创建topictemplate_detail表
CREATE TABLE IF NOT EXISTS topictemplate_detail (
  id int8 NOT NULL PRIMARY KEY,
  pid int8,
  templateid int8,
  position varchar(255),
  name varchar(255),
  type varchar(255),
  content varchar(255),
  isdefault bool,
  modulestyle varchar(255),
  weight float8
);
-- ----------------------------
-- Records of topictemplate_detail
-- ----------------------------
BEGIN;
INSERT INTO topictemplate_detail VALUES (1, NULL, 1, '1', NULL, NULL, NULL, NULL, NULL, NULL);
INSERT INTO topictemplate_detail VALUES (2, NULL, 1, '2', NULL, NULL, NULL, NULL, NULL, NULL);
INSERT INTO topictemplate_detail VALUES (3, 1, 1, 'layerfilter', NULL, NULL, NULL, NULL, NULL, NULL);
COMMIT;
--创建topiccontenttemplate表
CREATE TABLE topiccontenttemplate (
  id int8 NOT NULL PRIMARY KEY,
  pid int8,
  name varchar(255),
  displayname varchar(255),
  thumbnail varchar(255),
  style text,
  nodetype int2,
  description varchar(255),
  type varchar(255),
  subtype varchar(255)
);
-- ----------------------------
-- Records of topiccontenttemplate
-- ----------------------------
BEGIN;
INSERT INTO topiccontenttemplate VALUES (50, NULL, 'encyblack', '黑色标题卡片', '1454', '{"gutter":2,"marginbottom":1,"pagemargin":2,"width":"300","height":"300","xs":24,"sm":24,"md":24,"lg":24,"xl":24,"children":{"borderRadius":"0.25rem","background":"#fff","fontSize":"1rem","fontWeight":"700","color":"rgba(0,0,0,0.75)"}}', 1, NULL, 'card', 'encyblack');
INSERT INTO topiccontenttemplate VALUES (16, 0, 'nostylemap', '无样式地图', NULL, NULL, 1, NULL, 'map', 'nostylemap');
INSERT INTO topiccontenttemplate VALUES (2, 4, 'layerfilterM', '图层选择-多选', '3843', '{"position":"absolute","top":"12px","left":"12px","width":"220px","height":"auto","fontSize":"14px","color":"rgba(0,0,0,0.75)","fontWeight":"700","borderRadius":"4px","zIndex":"1001"}', 1, '', 'layerfilter', 'layerfilterM');
INSERT INTO topiccontenttemplate VALUES (56, 8, 'tabmodel', '模板标签页', '', '{"gutter":16,"marginbottom":1,"children":{"default":{},"primary":{}}}', 1, NULL, 'tab', 'tabmodel');
INSERT INTO topiccontenttemplate VALUES (3, 0, 'tabTitle', '有标题标签页', NULL, '', 1, '', 'tab', 'tabTitle');
INSERT INTO topiccontenttemplate VALUES (9, 0, 'tabNotitle', '无标题标签页', NULL, NULL, 1, NULL, 'tab', 'tabNotitle');
INSERT INTO topiccontenttemplate VALUES (57, 8, 'tabdynamic', '动态标签页', '1546', '{"gutter":16,"children":{"default":{"border":"1px solid #dcdfe6","margin":"0.5rem 0.5rem 0.5rem 0 ","backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","borderRadius":"0.25rem","fontSize":"0.875rem","height":"2rem"},"primary":{"border":"1px solid #3c78dc","margin":"0.5rem 0.5rem 0.5rem 0 ","backgroundColor":"#3c78dc","color":"#fff","borderRadius":"0.25rem","fontSize":"0.875rem","height":"2rem"}},"marginbottom":1}', 1, NULL, 'tab', 'tabdynamic');
INSERT INTO topiccontenttemplate VALUES (48, 47, 'blacktitle', '黑色标题', NULL, NULL, 1, NULL, 'blacktitle', 'blacktitle');
INSERT INTO topiccontenttemplate VALUES (53, 52, 'importpan', '重点关注面板', NULL, NULL, 1, NULL, 'importpan', 'importpan');
INSERT INTO topiccontenttemplate VALUES (45, NULL, 'other', '其他组件', NULL, NULL, 0, NULL, 'other', 'other');
INSERT INTO topiccontenttemplate VALUES (47, NULL, 'title', '标题', NULL, NULL, 0, NULL, 'title', 'title');
INSERT INTO topiccontenttemplate VALUES (52, NULL, 'important', '重点关注', NULL, NULL, 0, NULL, 'important', 'important');
INSERT INTO topiccontenttemplate VALUES (49, 8, 'tabblock', '底部按钮标签页', '1543', '{"gutter":16,"children":{"default":{"border":"1px solid #dcdfe6","margin":"0.5rem 0.5rem 0.5rem 0 ","backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","borderRadius":"0.25rem","fontSize":"0.875rem","height":"2rem"},"primary":{"border":"1px solid #3c78dc","margin":"0.5rem 0.5rem 0.5rem 0 ","backgroundColor":"#3c78dc","color":"#fff","borderRadius":"0.25rem","fontSize":"0.875rem","height":"2rem"}},"marginbottom":1}', 1, NULL, 'tab', 'tabblock');
INSERT INTO topiccontenttemplate VALUES (17, 10, 'title', '标题', NULL, NULL, 1, NULL, 'title', 'title');
INSERT INTO topiccontenttemplate VALUES (21, 13, 'topicML', '展板', NULL, NULL, 1, NULL, 'topicML', 'topicML');
INSERT INTO topiccontenttemplate VALUES (22, 14, 'mobile', '移动端模板', NULL, NULL, 1, NULL, 'mobile', 'mobile');
INSERT INTO topiccontenttemplate VALUES (15, 26, 'card', '普通卡片', NULL, NULL, 1, NULL, 'card', 'card');
INSERT INTO topiccontenttemplate VALUES (32, 0, 'encycard', '百科卡片', NULL, NULL, 1, NULL, 'card', 'encycard');
INSERT INTO topiccontenttemplate VALUES (11, 0, NULL, '标题布局页', NULL, NULL, 0, NULL, NULL, NULL);
INSERT INTO topiccontenttemplate VALUES (18, 0, 'leftTab', '左侧标签页', NULL, NULL, 1, NULL, 'tab', 'leftTab');
INSERT INTO topiccontenttemplate VALUES (37, 0, 'tabLdecoration', '左侧下划线标签页', NULL, NULL, 1, NULL, 'tab', 'tabLdecoration');
INSERT INTO topiccontenttemplate VALUES (39, 0, 'tabRdecoration', '右侧下划线标签页', NULL, NULL, 1, NULL, 'tab', 'tabRdecoration');
INSERT INTO topiccontenttemplate VALUES (10, 0, '', '标准布局页', NULL, NULL, 0, NULL, '', '');
INSERT INTO topiccontenttemplate VALUES (27, 8, 'tabbtnT', '顶部按钮标签页', '1546', '{"gutter":16,"children":{"default":{"border":"1px solid #dcdfe6","margin":"0.5rem 0.5rem 0.5rem 0 ","backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","borderRadius":"0.25rem","fontSize":"0.875rem","height":"2rem"},"primary":{"border":"1px solid #3c78dc","margin":"0.5rem 0.5rem 0.5rem 0 ","backgroundColor":"#3c78dc","color":"#fff","borderRadius":"0.25rem","fontSize":"0.875rem","height":"2rem"}},"marginbottom":1}', 1, NULL, 'tab', 'tabbtnT');
INSERT INTO topiccontenttemplate VALUES (19, 0, 'yqmap', '疫情地图', NULL, NULL, 1, NULL, 'map', 'yqmap');
INSERT INTO topiccontenttemplate VALUES (23, 0, 'yqfxmap', '疫情风险地图', NULL, NULL, 1, NULL, 'map', 'yqfxmap');
INSERT INTO topiccontenttemplate VALUES (24, 6, 'topicmap', '专题地图', NULL, '{"height":"100%","width":"100%","lat":35.26,"lng":116.35,"level":14,"position":{}}', 1, NULL, 'map', 'topicmap');
INSERT INTO topiccontenttemplate VALUES (25, 26, 'spacecard', '空间卡片', NULL, NULL, 1, NULL, 'card', 'spacecard');
INSERT INTO topiccontenttemplate VALUES (34, 33, 'goodafternoon', '下午好！', NULL, NULL, 1, '欢迎来到分析决策平台，每天都要元气满满哦！无论上班下班，我都会一直为您服务！', 'goodafternoon', 'goodafternoon');
INSERT INTO topiccontenttemplate VALUES (46, 45, 'newstatu', '最新动态', NULL, NULL, 1, NULL, 'newstatu', 'newstatu');
INSERT INTO topiccontenttemplate VALUES (12, NULL, 'ency', '百科', NULL, NULL, 0, NULL, 'ency', 'ency');
INSERT INTO topiccontenttemplate VALUES (26, NULL, 'card', '卡片', NULL, NULL, 0, NULL, 'card', 'card');
INSERT INTO topiccontenttemplate VALUES (30, NULL, 'encyclope', '百度百科', NULL, NULL, 1, NULL, 'encyclope', 'encyclope');
INSERT INTO topiccontenttemplate VALUES (20, NULL, 'topicC', '专题中', NULL, NULL, 1, NULL, 'topicC', 'topicC');
INSERT INTO topiccontenttemplate VALUES (33, NULL, 'helloword', '问候语', NULL, NULL, 0, NULL, 'helloword', 'helloword');
INSERT INTO topiccontenttemplate VALUES (43, NULL, 'Carousel', '走马灯', NULL, NULL, 0, NULL, 'Carousel', 'Carousel');
INSERT INTO topiccontenttemplate VALUES (31, NULL, 'encylist', '百科列表', NULL, NULL, 1, NULL, 'encylist', 'encylist');
INSERT INTO topiccontenttemplate VALUES (36, NULL, 'tabRTdecoration', '右上角下划线标签页', '1544', NULL, 1, NULL, 'tab', 'tabRTdecoration');
INSERT INTO topiccontenttemplate VALUES (35, 8, 'tabLTdecoration', '左上角下划线标签页', '1545', '{"gutter":16,"marginbottom":1,"children":{"primary":{"fontSize":"0.875rem","color":"#3C78DC","height":"2rem","borderBottom":"2px solid #3C78DC","margin":"0.5rem 1rem 0.5rem 0","lineHeight":"2rem"},"default":{"fontSize":"0.875rem","fontWeight":400,"color":"rgba(0,0,0,0.45)","height":"2rem","borderBottom":"2px solid #ffffff","margin":"0.5rem 1rem 0.5rem 0","lineHeight":"2rem"}}}', 1, NULL, 'tab', 'tabLTdecoration');
INSERT INTO topiccontenttemplate VALUES (29, NULL, 'tabbtnL', '左侧按钮标签页', '1547', '{"gutter":2,"marginbottom":1,"width":"300","height":"300","span":24,"children":{"primary":{"fontSize":"0.875rem","color":"#fff","backgroundColor":"#3c78dc","borderRadius":"0.25rem","height":"2rem"},"default":{"fontSize":"0.875rem","color":"rgba(0,0,0,0.75)","backgroundColor":"#fff","borderRadius":"0.25rem","height":"2rem"}}}', 1, NULL, 'tab', 'tabbtnL');
INSERT INTO topiccontenttemplate VALUES (28, NULL, 'tabbtnR', '右侧按钮标签页', '1548', NULL, 1, NULL, 'tab', 'tabbtnR');
INSERT INTO topiccontenttemplate VALUES (41, 40, 'indexone', '指标模板', NULL, NULL, 1, NULL, 'index', 'indexone');
INSERT INTO topiccontenttemplate VALUES (42, 40, 'indextwo', '指标模板', NULL, NULL, 1, NULL, 'index', 'indextwo');
INSERT INTO topiccontenttemplate VALUES (44, 43, 'carouselone', '走马灯模板', NULL, NULL, 1, NULL, 'carouselone', 'carouselone');
INSERT INTO topiccontenttemplate VALUES (6, NULL, 'map', '地图', NULL, NULL, 0, NULL, 'map', 'map');
INSERT INTO topiccontenttemplate VALUES (4, NULL, 'layerfilter', '图层筛选', NULL, NULL, 0, NULL, 'layerfilter', 'layerfilter');
INSERT INTO topiccontenttemplate VALUES (8, NULL, 'layout', '布局', NULL, NULL, 0, NULL, 'layout', 'layout');
INSERT INTO topiccontenttemplate VALUES (54, 8, 'tabmulti', '复合型标签页', '2490', '{"gutter":16,"marginbottom":1,"children":{"default":{},"primary":{}}}', 1, NULL, 'tab', 'tabmulti');
INSERT INTO topiccontenttemplate VALUES (40, NULL, 'indexmodel', '指标', NULL, NULL, 0, NULL, 'index', 'indexmodel');
INSERT INTO topiccontenttemplate VALUES (7, 0, 'layerfilterS', '图层筛选-单选', NULL, NULL, 1, NULL, 'layerfilter', 'layerfilterS');
INSERT INTO topiccontenttemplate VALUES (1, 0, 'map1', '地图标准', NULL, NULL, 1, NULL, 'map', 'map1');
INSERT INTO topiccontenttemplate VALUES (5, 0, 'map2', '地图简洁', NULL, NULL, 1, NULL, 'map', 'map2');
INSERT INTO topiccontenttemplate VALUES (51, 12, 'whitecard', '空白卡片', '1436', '{"gutter":16,"marginbottom":1,"pagemargin":0,"xs":24,"sm":24,"md":24,"lg":24,"xl":24,"children":{"padding":"0.75rem","borderRadius":"0.25rem","background":"#fff","height":""}}', 1, NULL, 'card', 'whitecard');
COMMIT;
```



### 4.6 画像接口

画像接口需要依赖**文件资源管理接口**。

画像接口需要以下数据表：***portrayal***、***portrayal_detail***、***portrayaltemplate***、***portrayaltemplate_detail***表，这些数据表的创建SQL如下：

```sql
--创建portrayal表
CREATE TABLE IF NOT EXISTS portrayal (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  style text,
  creator int8,
  creattime timestamp(6),
  updatetime timestamp(6),
  thumbnail int8,
  copyby int8,
  templateid int8,
  ispublic bool,
  type int2,
  isdelete int2,
  becalled int4,
  updator int8,
  platform int8
);
--创建portrayal_detail表
CREATE TABLE IF NOT EXISTS portrayal_detail (
  id int8 NOT NULL PRIMARY KEY,
  portrayalid int8,
  position varchar(255),
  content varchar(255),
  contenttype varchar(255),
  style text,
  col int2,
  row int2,
  weight int4 NOT NULL,
  pid int8,
  name varchar(255),
  displayname varchar(255)
);
--创建portrayaltemplate表
CREATE TABLE IF NOT EXISTS portrayaltemplate (
  id int8 NOT NULL PRIMARY KEY,
  pid int8,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  style text,
  thumbnail int8,
  type int4
);
-- ----------------------------
-- Records of portrayaltemplate
-- ----------------------------
BEGIN;
INSERT INTO portrayaltemplate VALUES (2, NULL, 't2_c3', '二标三图', '四行（包括2个标题3个图表）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 42, 1);
INSERT INTO portrayaltemplate VALUES (3, NULL, 'only_t2', '仅二图', '一行（2列图表）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 43, 1);
INSERT INTO portrayaltemplate VALUES (5, NULL, 't2_c2(1)', '二标二图(1)', '4行（包括2个标题，两个图表）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 41, 1);
INSERT INTO portrayaltemplate VALUES (6, NULL, 't2_c2(2)', '二标二图(2)', '4行（包括2个标题，两个图表）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 45, 1);
INSERT INTO portrayaltemplate VALUES (1, NULL, '垂直瀑布流', '自定义布局', '自定义页面布局样式', '{"padding":8,"marginrow":8,"gutter":8}', 281, 0);
INSERT INTO portrayaltemplate VALUES (4, NULL, 't1_c1_table1', '一标一图一表', '三行（包括一个标题，一个图表，一个表格）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 44, 1);
INSERT INTO portrayaltemplate VALUES (10, 100, 'c1', '一图', '一行', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 48, 1);
INSERT INTO portrayaltemplate VALUES (11, 100, 't4_c4', '四标四图', '八行', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 49, 1);
INSERT INTO portrayaltemplate VALUES (9, 100, 't3_c3', '三标三图', '六行（包括3个标题3个图表）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 47, 1);
INSERT INTO portrayaltemplate VALUES (7, 100, 't3_table(2)', '三标二表', '五行（包含三个标题，两个表格）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 41, 1);
INSERT INTO portrayaltemplate VALUES (8, 100, 't4_table_c2', '疫情分析画像', '七行（包含4个标题一个表格2个图表）', '{"height":800,"width":500,"padding":8,"marginrow":8,"gutter":8}', 41, 1);
COMMIT;
--创建portrayaltemplate_detail表
CREATE TABLE IF NOT EXISTS portrayaltemplate_detail (
  id int8 NOT NULL PRIMARY KEY,
  portrayaltemplateid int8,
  position varchar(255),
  content varchar(255),
  contenttype varchar(255),
  style text,
  col int2,
  row int2
);
-- ----------------------------
-- Records of portrayaltemplate_detail
-- ----------------------------
BEGIN;
INSERT INTO portrayaltemplate_detail VALUES (1, 4, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (2, 4, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (3, 4, '3', NULL, 'graph', '{"span":24,"height":320,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (13, 5, '3', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (11, 5, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (14, 5, '4', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (12, 5, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (25, 8, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (17, 6, '3', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (16, 6, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (15, 6, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (26, 8, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (27, 8, '3', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (18, 6, '4', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (28, 8, '4', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (29, 8, '5', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 5);
INSERT INTO portrayaltemplate_detail VALUES (30, 8, '6', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 6);
INSERT INTO portrayaltemplate_detail VALUES (31, 8, '7', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 7);
INSERT INTO portrayaltemplate_detail VALUES (32, 9, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (23, 7, '4', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (24, 7, '5', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 5);
INSERT INTO portrayaltemplate_detail VALUES (20, 7, '1', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (21, 7, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (22, 7, '3', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (34, 9, '3', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (36, 9, '5', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 5);
INSERT INTO portrayaltemplate_detail VALUES (5, 2, '2', NULL, 'graph', '{"span":12,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (6, 2, '3', NULL, 'graph', '{"span":12,"height":220,"width":""}', 2, 2);
INSERT INTO portrayaltemplate_detail VALUES (4, 2, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (7, 2, '4', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (8, 2, '5', NULL, 'graph', '{"span":24,"height":320,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (10, 3, '2', NULL, 'graph', '{"span":12,"height":220,"width":""}', 2, 1);
INSERT INTO portrayaltemplate_detail VALUES (9, 3, '1', NULL, 'graph', '{"span":12,"height":220,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (33, 9, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (35, 9, '4', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (37, 9, '6', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 6);
INSERT INTO portrayaltemplate_detail VALUES (38, 11, '1', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 1);
INSERT INTO portrayaltemplate_detail VALUES (39, 11, '2', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 2);
INSERT INTO portrayaltemplate_detail VALUES (40, 11, '3', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 3);
INSERT INTO portrayaltemplate_detail VALUES (41, 11, '4', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 4);
INSERT INTO portrayaltemplate_detail VALUES (42, 11, '5', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 5);
INSERT INTO portrayaltemplate_detail VALUES (43, 11, '6', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 6);
INSERT INTO portrayaltemplate_detail VALUES (44, 11, '7', NULL, 'graph', '{"span":24,"height":32,"width":""}', 1, 7);
INSERT INTO portrayaltemplate_detail VALUES (45, 11, '8', NULL, 'graph', '{"span":24,"height":220,"width":""}', 1, 8);
COMMIT;
```



### 4.7 图表接口

图表接口依赖**文件资源管理接口**、**专题配置接口**和**画像接口**。

图表接口需要以下数据表：***graphmeta***、***graphtemplate***、***graphtemplate_detail***表，这些数据表的创建SQL如下：

```sql
--创建graphmeta表
CREATE TABLE IF NOT EXISTS graphmeta (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  graphtype varchar(255),
  datasource text,
  datasourcetype int4,
  style text,
  params text,
  creattime timestamp(6),
  updatetime timestamp(6),
  creator int8,
  updator int8,
  creattype int2,
  becalled int4,
  isdelete bool,
  graphtemplateid int8,
  copyby int8,
  pageviews int8,
  weight int8,
  thumbnail int8,
  type varchar(255),
  filemetaid int8 DEFAULT 0,
  isdrilling int4 DEFAULT 0,
  interaction text,
  platform int8
);
--创建graphtemplate表
CREATE TABLE IF NOT EXISTS graphtemplate (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  graphtype varchar(255),
  datasource text,
  datasourcetype int4,
  style text,
  params text,
  creattime timestamp(6),
  inproperty bool,
  pid int8,
  thumbnail int8,
  icon varchar(255),
  icontype int2,
  weight float8,
  type varchar(255),
  interaction text,
  formattype int2
);
-- ----------------------------
-- Records of graphtemplate
-- ----------------------------
BEGIN;
INSERT INTO graphtemplate VALUES (11, 'table', '表格', '表格', 'table', '', NULL, '', NULL, '2020-09-22 16:57:02', NULL, NULL, 37, 'icon-table', 1, 1, 'table', NULL, NULL);
INSERT INTO graphtemplate VALUES (66, 'othernav', '导航栏', '导航栏', 'othernav', '{year: 2020, month: 12,unit:"月"}', 1, '{"container":{"padding":"0px","margin":"0px"},"monthUnit":{"fontFamily":"PingFang SC","color":"rgba(0, 0, 0, 0.45)","fontSize":"12px","fontWeight":400},"wrapp":{"borderRadius":"4px","alignItems":"center","background":"rgba(32, 159, 132, 0.9)","flexDirection":"column","display":"flex","width":"52px","height":"58px"},"month":{"fontFamily":"PingFang SC","borderRadius":"4px","color":"rgba(0, 0, 0, 0.75)","textAlign":"center","background":"#FFFFFF","width":"44px","lineHeight":"34px","fontSize":"15px","fontWeight":"bold","height":"34px"},"year":{"fontFamily":"PingFang SC","color":"#FFFFFF","display":"flex","fontSize":"12px","justifyContent":"spaceAround","fontWeight":400},"yearLeftDot":{"marginRight":"4px"},"yearRightDot":{"marginLeft":"4px"}}', NULL, '2021-01-20 18:10:36', NULL, NULL, 2488, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (10, 't_center_angle', '三角标题', '标题', 'titleangle', '{"displayName":"文本内容"}', 1, '{"color":"#3C78DC","fontSize":"0.875rem","fontWeight":"400"}', NULL, '2020-10-12 13:38:18', NULL, 8, 30, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (63, 'otherprogress', '进度条', '进度条', 'otherprogress', '{"data":[{"districtName":"第一分局","year":"2019","children":[{"unit":"%","name":"规划","value":"42.3100","key":"planPercent"},{"unit":"%","name":"现状","value":"40.200","key":"planPercent"}],"displayName":"土地开发强度现状"}]}', 1, '{"plannum":{"fontFamily":"PingFang SC","color":"rgba(0, 0, 0, 0.45)","fontSize":"10px","fontWeight":400},"num":{"fontFamily":"PingFang SC","color":"rgba(1, 1, 1, 0.75)","fontSize":"14px","marginleft":"12px","fontWeight":"bold"},"planname":{"fontFamily":"PingFang SC","color":"rgba(0, 0, 0, 0.45)","fontSize":"10px","fontWeight":400},"title":{"fontFamily":"PingFang SC","color":"rgba(0, 0, 0, 0.75)","fontSize":"14px","marginleft":"20px","fontWeight":400},"body":{"backgroundColor":"#E9EEED","width":"153px","borderradius":"2px","marginleft":"20px","height":"12px"},"plandot":{"borderLeft":"1px dotted rgba(0, 0, 0, 0.45)","width":"1px","height":"19px"},"lightnum":{"backgroundColor":"#0a5bcc"}}', NULL, '2021-01-20 18:10:36', NULL, 42, 11246, NULL, NULL, NULL, 'other', NULL, 2);
INSERT INTO graphtemplate VALUES (4, 'graph', '统计图表', '统计图表', 'graph', NULL, NULL, '', NULL, NULL, NULL, NULL, 1, 'icon-chart', 1, 4, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (8, 'text', '文本', NULL, 'text', '', NULL, '', NULL, '2020-10-12 13:38:23', NULL, NULL, NULL, 'icon-text', 1, 2, 'text', NULL, NULL);
INSERT INTO graphtemplate VALUES (73, 'tabdynamic', '动态tab', '动态tab', 'tabdynamic', '{"text1":1521,"text2":205}', 1, '{"default":"中山市重大产业平台内存量用地,可用于招商引资的连片用地总规模约{text1=1521}亩,共{text2=1521}宗","t2style":{},"defstyle":{"color":"rgba(0, 0, 0, 0.75)","borderRadius":"8px","background":"#EEF7F5","height":"72px","fontSize":"14px","padding":"12px","display":"flex","alignItems":"center","justifyContent":"center","width":"100%"},"t1style":{}}', NULL, '2021-03-25 15:07:26', NULL, 4444, 12312, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (7, 'indexsin', '单项指标', NULL, 'indexsin', '', NULL, '', NULL, NULL, NULL, 4444, 39, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (22, 'pie', '饼图', '饼图', 'pie', '{"dataset":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}],"dimensions":["name","value"]}', 1, '{"color":["#edbcb9 ","#e08e8b ","#d46161 ","#c73c40 ","#b91923 "],"legend":{"top":"top","itemHeight":12,"itemWidth":16,"textStyle":{"color":"rgba(0,0,0,0.7)","fontSize":14}},"series":[{"center":["50%","58%"],"type":"pie","radius":"70%"}],"tooltip":{},"title":{"left":"","text":""},"dataset":{"source":[],"dimensions":[]}}', '', '2019-11-07 11:27:24', NULL, 4, 38813, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (30, 'pierose', '玫瑰图', '统计图表', 'pierose', '{"metadata":[{"displayname":"","fields":[],"dimension":["name","value"]}],"code":0,"data":[{"name":"白云区","value":13},{"name":"海珠区","value":20},{"name":"天河区","value":17},{"name":"越秀区","value":31},{"name":"黄埔区","value":6},{"name":"荔湾","value":17},{"name":"番禺区","value":7},{"name":"花都区","value":8},{"name":"增城区","value":6},{"name":"从化区","value":5},{"name":"南沙区","value":4},{"name":"萝岗区","value":3}],"count":1,"message":""}', 1, '{"color":["#90bcf5","#42CFA0","#9186CF","#D9AF80","#5BCAD4","#559BE6","#D97E87","#E6906C","#D9C57E","#BBB0C6"],"dataset":{},"series":[{"center":["50%","50%"],"roseType":"area","name":"面积模式","type":"pie","radius":["30%","80%"]}],"tooltip":{"trigger":"item"},"label":{"formatter":"{b}: {d}%","fontSize":14},"title":{"padding":[125,0,0,0],"subtext":"药店总量","x":"center","text":"137"}}', NULL, '2020-04-10 17:44:28', NULL, 4, 247, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (9, 't_center_circle', '圆形标题', '标题', 'titlecircle', '{"displayName":"文本内容"}', 1, '{"color":"#fff","fontSize":"0.875rem","fontWeight":"400"}', NULL, '2020-10-12 13:38:21', NULL, 8, 29, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (27, 'gaugemulti', '多个仪表盘', '多个仪表盘', 'gaugemulti', '{"code":0,"count":6,"message":"","data":[{"name":"新增确诊","weight":1,"value":1},{"name":"累计确诊","weight":2,"value":1333},{"name":"治愈","weight":3,"value":664},{"name":"治愈率","weight":4,"value":49.81},{"name":"死亡","weight":5,"value":5},{"name":"死亡率","weight":6,"value":0.38}]}', 1, '{"backgroundColor":"","series":[{}]}', NULL, '2020-02-20 17:24:10', NULL, 4, 298, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (17, 'radar', '雷达图', '雷达图', 'radar', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value1","value2"]}],"data":[{"name":"安全","value1":339,"value2":112},{"name":"共享","value1":339,"value2":3},{"name":"开放","value1":339,"value2":0},{"name":"绿色","value1":339,"value2":20},{"name":"协调","value1":339,"value2":10},{"name":"创新","value1":343,"value2":43}]}', 1, '{"color":["#69dbb1","#90bcf5"],"tooltip":{},"radar":{"splitNumber":4,"center":["50%","50%"],"radius":"60%","name":{"textStyle":{"color":"#333","borderRadius":100,"padding":[0,0],"fontSize":"16"}},"startAngle":"0","indicator":[],"splitArea":{"show":true,"areaStyle":{"color":["#EFF5FD","#E1EBFB"]}},"axisLine":{"lineStyle":{"color":"#D0DFF4"}},"splitLine":{"lineStyle":{"color":"transparent","width":1}}},"series":[{"name":"","type":"radar","data":[{"value":[],"name":"今年","areaStyle":{"normal":{"color":"#69dbb1","opacity":"0.4"}}},{"value":[],"name":"2018","areaStyle":{"normal":{"color":"#90bcf5","opacity":"0.4"}}}]}]}', '', '2019-11-07 11:27:24', NULL, 4, 38808, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (19, 'liquidfill', '动态水球图', '动态水球图', 'liquidfill', '', 1, '', '', '2019-11-07 11:27:24', NULL, 4444, 23, NULL, NULL, NULL, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (29, 't_left_curve', '趋势标题', '标题', 'titlecurve', '{"displayName":"文本内容"}', 1, '{"color":"#3C78DC","fontSize":"0.875rem","fontWeight":"400"}', NULL, '2020-10-12 13:38:04', NULL, 8, 299, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (41, 'barhboth-d', '双向条形图', '统计图表', 'bar', '/api/ncovepidemic/auxiliaryrisk/agepercent', 0, '{"tooltip":{"show":true},"grid":[{"show":false,"left":"4%","top":10,"bottom":10,"containLabel":true,"width":"46%"},{"show":false,"left":"51.5%","top":30,"bottom":10,"width":"0%"},{"show":false,"right":"4%","top":10,"bottom":10,"containLabel":true,"width":"46%"}],"dataset":{"source":[],"dimensions":[]},"xAxis":[{"type":"value","inverse":true,"axisLine":{"show":false},"axisTick":{"show":false},"position":"top","axisLabel":{"show":true,"textStyle":{"color":"rgba(255,255,255,0.8)","fontSize":12}},"splitLine":{"show":true,"lineStyle":{"color":"rgba(255,255,255,0.06)","width":1,"type":"solid"}}},{"gridIndex":1,"show":false},{"gridIndex":2,"type":"value","axisLine":{"show":false},"axisTick":{"show":false},"position":"top","axisLabel":{"show":true,"textStyle":{"color":"rgba(255,255,255,0.8)","fontSize":12}},"splitLine":{"show":true,"lineStyle":{"color":"rgba(255,255,255,0.06)","width":1,"type":"solid"}}}],"yAxis":[{"type":"category","inverse":true,"position":"right","axisLine":{"show":false},"axisTick":{"show":false},"axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0)","fontSize":6}}},{"gridIndex":1,"type":"category","inverse":true,"position":"left","axisLine":{"show":false},"axisTick":{"show":false},"axisLabel":{"show":true,"textStyle":{"color":"rgba(255,255,255,0.8)","fontSize":12,"align":"center"}},"data":["81岁以上","71-80岁","61-70岁","51-60岁","41-50岁","31-40岁","21-30岁","15-20岁","11-14岁","4-10岁","3岁以下"]},{"gridIndex":2,"type":"category","inverse":true,"position":"left","axisLine":{"show":false},"axisTick":{"show":false},"axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0)","fontSize":6}}}],"series":[{"name":"患者年龄","type":"bar","barGap":20,"barWidth":15,"label":{"normal":{"show":false},"emphasis":{"show":true,"position":"left","offset":[0,0],"textStyle":{"color":"#fff","fontSize":14}}},"itemStyle":{"normal":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#3c8cf0"},{"offset":1,"color":"#50dcff"}]}},"emphasis":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#3c8cf0"},{"offset":1,"color":"#50dcff"}]}}}},{"name":"患者年龄","type":"bar","barGap":20,"barWidth":15,"xAxisIndex":2,"yAxisIndex":2,"label":{"normal":{"show":false},"emphasis":{"show":true,"position":"right","offset":[0,0],"textStyle":{"color":"#fff","fontSize":14}}},"itemStyle":{"normal":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#50dcff"},{"offset":1,"color":"#3c8cf0"}]}},"emphasis":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#50dcff"},{"offset":1,"color":"#3c8cf0"}]}}}}]}', NULL, '2020-04-13 10:10:50', NULL, 0, 577, NULL, NULL, NULL, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (32, 'bargender', '性别比例', '统计图表', 'bargender', '{"dataset":[131,113]}', 1, '{"dataset":{"dimension":["name","male","female"],"source":[{"name":"性别比例","male":"","female":""}]},"xAxis":[{"type":"value","show":false,"max":461}],"yAxis":[{"type":"category","show":false}],"color":["#618cff","#ffa57c","#8f949a"],"series":[{"name":"男","type":"bar","stack":"性别比例","barWidth":16,"label":{"normal":{"show":true,"offset":[0,0],"formatter":"{male|}\n{b| }\n{c|311人}","rich":{"male":{"height":40,"align":"center","backgroundColor":{"image":"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABQAAAAoCAYAAAD+MdrbAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAyZpVFh0WE1MOmNvbS5hZG9iZS54bXAAAAAAADw/eHBhY2tldCBiZWdpbj0i77u/IiBpZD0iVzVNME1wQ2VoaUh6cmVTek5UY3prYzlkIj8+IDx4OnhtcG1ldGEgeG1sbnM6eD0iYWRvYmU6bnM6bWV0YS8iIHg6eG1wdGs9IkFkb2JlIFhNUCBDb3JlIDUuNS1jMDE0IDc5LjE1MTQ4MSwgMjAxMy8wMy8xMy0xMjowOToxNSAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNjcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wTU09Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9tbS8iIHhtbG5zOnN0UmVmPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvc1R5cGUvUmVzb3VyY2VSZWYjIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtcE1NOkRvY3VtZW50SUQ9InhtcC5kaWQ6ODkzRDIwQTE0MTIxMTFFODkyOTU4RUU5NzM3MjE3MDMiIHhtcE1NOkluc3RhbmNlSUQ9InhtcC5paWQ6ODkzRDIwQTA0MTIxMTFFODkyOTU4RUU5NzM3MjE3MDMiIHhtcDpDcmVhdG9yVG9vbD0iQWRvYmUgUGhvdG9zaG9wIENDIDIwMTUgKFdpbmRvd3MpIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0UmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6NTQ4MERFMjNCRDNDMTFFNzgyQTFFRkM1MDA3MjdBRTYiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6NTQ4MERFMjRCRDNDMTFFNzgyQTFFRkM1MDA3MjdBRTYiLz4gPC9yZGY6RGVzY3JpcHRpb24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIiPz4iuAMWAAABl0lEQVR42uzWTSgEYRzH8ZmJcPDOkRMXG1cl5SInJ1e5aEtuLhwoNy5K0mpzWCXJyQEH5eDEwUVecuKCKEJ5qU2L8R2e1fT0zLM7O3NR869P284zz69nnmeemTFt2zac6kz+/kpVjG6MoB0ZbGMWJ/jKnrg3bP78WoZ3OW2DWEcPKlGHAWyiC6aqk1fFMI4yRVsDJlDrJ7AXjZp2Z4RtfgLrDX0VocJP4H2OwE+8+Ancwa2mfR9nfgIPMYMnxchuMI071TzoKoFHxMXKOjfrERaw6zWxuvrACjZQLW5kZ27fdSslT0ETyqXjGdeuqHH1e8WFe8e4A0vQL7ZZs2oXSOVc/jnmsJodtTuwDynDX7WKPmmsyas8ahReY6rbpiVAYEwVaBkhlCVNcqFlhzoqI+zLjAL/ygyQY4Z9H1qqkKsAgdeqwESAwHnV4yslnnVD4oWeTz1gEUuqwGdMYQtV4tgkOjzCklgWnyRprye203Dg+h/XjO5YOjevldW9c0qjrRcF/tdA029brsBT8aEke8OlqsO3AAMAxyBOvxLL2/sAAAAASUVORK5CYII="}},"b":{"fontSize":18,"align":"center","padding":[40,-30,-10,-40]},"c":{"fontSize":18,"align":"center"}}}},"itemStyle":{"barBorderRadius":10}},{"name":"女","type":"bar","stack":"性别比例","barWidth":16,"label":{"normal":{"show":true,"offset":[0,0],"formatter":"{Female|}\n{b| }\n{c|150人}","rich":{"Female":{"height":40,"align":"center","backgroundColor":{"image":"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAdCAYAAACaCl3kAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAyZpVFh0WE1MOmNvbS5hZG9iZS54bXAAAAAAADw/eHBhY2tldCBiZWdpbj0i77u/IiBpZD0iVzVNME1wQ2VoaUh6cmVTek5UY3prYzlkIj8+IDx4OnhtcG1ldGEgeG1sbnM6eD0iYWRvYmU6bnM6bWV0YS8iIHg6eG1wdGs9IkFkb2JlIFhNUCBDb3JlIDUuNS1jMDE0IDc5LjE1MTQ4MSwgMjAxMy8wMy8xMy0xMjowOToxNSAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNjcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wTU09Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9tbS8iIHhtbG5zOnN0UmVmPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvc1R5cGUvUmVzb3VyY2VSZWYjIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtcE1NOkRvY3VtZW50SUQ9InhtcC5kaWQ6OTgxQUVBQkE0MTIwMTFFODlBRjc4REM5QkNCNEQ3NkEiIHhtcE1NOkluc3RhbmNlSUQ9InhtcC5paWQ6OTgxQUVBQjk0MTIwMTFFODlBRjc4REM5QkNCNEQ3NkEiIHhtcDpDcmVhdG9yVG9vbD0iQWRvYmUgUGhvdG9zaG9wIENDIDIwMTUgKFdpbmRvd3MpIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0UmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6NTQ4MERFMjNCRDNDMTFFNzgyQTFFRkM1MDA3MjdBRTYiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6NTQ4MERFMjRCRDNDMTFFNzgyQTFFRkM1MDA3MjdBRTYiLz4gPC9yZGY6RGVzY3JpcHRpb24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIiPz5Op4glAAABOUlEQVR42mL8//8/Awj8zY1lgAJ2IG4C4lQg/gbEy4C4Foh/giSZJy8GK2JhwAQzgDgByhYE4lIgFgXiRGRFTGiaQArjsBgWC5XDqVEcixjYhVA5nBrvA/F7LBrfQeVwagQFQDEorJDEQOwSWODAALbAmQ/EN5ACaAEQH0dXxPgnJ4aBHIAtILYB8X80vJWQRiEgdsFimCtUDqfGACBmxaKRFSqHU2M4Hm+F49IoAsROeDQ6QdVgaAzEET3IUReITWMYEbEQhq5RDIgdidDoCFUL1xgETciEADNULVxjGAmJJgymUQKI7UjQaA/SA9JoRaQzkcPFCkScQs8yBABI7SmQxidQDz8gQtMDqNonTEg5QhGUzYC4A4uGDqicIlQtAwusuEMrIrHHBZJaJgYywRDX+IkYMYAAAwB6sjfXWpdRXAAAAABJRU5ErkJggg=="}},"b":{"fontSize":18,"fontWeight":100,"align":"center","padding":[40,0,-10,0]},"c":{"fontSize":18,"fontWeight":100,"align":"center"}}}},"itemStyle":{"barBorderRadius":10}}]}', NULL, NULL, NULL, 4, 245, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (18, 'tree', '树图', '树图', 'tree', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value"]}],"data":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}]}', 1, '{"color":["#90bcf5","#42CFA0","#9186CF","#D9AF80","#5BCAD4","#559BE6","#D97E87","#E6906C","#D9C57E","#BBB0C6"],"tooltip":{"trigger":"item","formatter":"{b}"},"series":[{"type":"treemap","width":"90%","height":"80%","top":"8%","roam":false,"nodeClick":false,"breadcrumb":{"show":false},"label":{"normal":{"show":true,"position":["10%","40%"]}},"itemStyle":{"normal":{"show":true,"textStyle":{"color":"#fff","fontSize":16},"borderWidth":1,"borderColor":"#fff"},"emphasis":{"label":{"show":true}}},"data":[]}]}', '', '2019-11-07 11:27:24', NULL, 4, 1418, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (67, 'otheranchor', '锚点', '锚点', 'otheranchor', '/api/topicconfig/preview/871', 0, '{"anchordom":{}}', NULL, '2021-01-22 09:16:29', NULL, 42, 11242, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (14, 'table', '表格', NULL, 'table', '/api/topicanalysis/epidemic/totalnum?num=10', 0, '{"border":true,"body1":{"backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","fontWeight":400,"height":"40px"},"body2":{"backgroundColor":"#fafafa","color":"rgba(0,0,0,0.75)","fontWeight":400,"height":"40px"},"header":{"margin":16,"backgroundColor":"#e8f4ff","color":"rgba(0,0,0,0.75)","height":"40px"},"tablestyle":{"backgroundColor":"#fff","height":"240px"}}', NULL, '2020-09-22 16:55:31', NULL, 0, 37, NULL, NULL, NULL, 'table', NULL, NULL);
INSERT INTO graphtemplate VALUES (68, 'mobileChoose', '移动模板选择器', '移动模板选择器', 'mobileChoose', '{"classname":"choosed","children":[{"default":"icon-pos","children":[],"displayname":"","type":"iconOp","key":"icon","dataSourceType":0},{"times":2,"default":"100000","children":[],"meta":"districtId","displayname":"","id":442000,"type":"codeOp","key":"code","dataSourceType":1}],"displayname":"模板选择器"}', 1, '{"codeOP":{"border":"0.0625rem solid rgba(255,255,255,0)","backgroundColor":"rgba(255,255,255,0)","fontFamily":"PingFang SC","color":"#FFFFFF","fontSize":"16px","lineHeight":"40px","fontWeight":400},"typeOP":{"marginRight":"20px","width":"80px"},"icon":{"color":"#FFFFFF","textAlign":"center","name":"icon-pos","width":"18px","fontSize":"18px","marginLeft":"15px"},"yearOP":{"marginRight":"20px","width":"100px"},"choosed":{"alignItems":"center","background":"#209F84","display":"flex","height":"48px"},"subChoosed":{"alignItems":"center","background":"#fff","display":"flex","height":"48px"}}', NULL, '2020-12-29 10:27:08', NULL, 42, NULL, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (31, 'barhboth', '双向条形图', '统计图表', 'barhboth', '{"code":0,"message":"","count":2,"metadata":[{"fields":[{"fieldkey":"taskarea","fieldname":"任务面积"},{"fieldkey":"name","fieldname":"地点"}],"dimension":["name","value"]}],"data":[{"name":"医院","value":10800},{"name":"学校","value":13500}]}', 1, '{"tooltip":{"show":true},"grid":[{"show":false,"left":"4%","top":10,"bottom":10,"containLabel":true,"width":"46%"},{"show":false,"left":"51.5%","top":30,"bottom":10,"width":"0%"},{"show":false,"right":"4%","top":10,"bottom":10,"containLabel":true,"width":"46%"}],"dataset":{"source":[],"dimensions":[]},"xAxis":[{"type":"value","inverse":true,"axisLine":{"show":false},"axisTick":{"show":false},"position":"top","axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0.8)","fontSize":12}},"splitLine":{"show":true,"lineStyle":{"color":"rgba(0,0,0,0.06)","width":1,"type":"solid"}}},{"gridIndex":1,"show":false},{"gridIndex":2,"type":"value","axisLine":{"show":false},"axisTick":{"show":false},"position":"top","axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0.8)","fontSize":12}},"splitLine":{"show":true,"lineStyle":{"color":"rgba(0,0,0,0.06)","width":1,"type":"solid"}}}],"yAxis":[{"type":"category","inverse":true,"position":"right","axisLine":{"show":false},"axisTick":{"show":false},"axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0)","fontSize":6}}},{"gridIndex":1,"type":"category","inverse":true,"position":"left","axisLine":{"show":false},"axisTick":{"show":false},"axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0.8)","fontSize":12,"align":"center"}},"data":[]},{"gridIndex":2,"type":"category","inverse":true,"position":"left","axisLine":{"show":false},"axisTick":{"show":false},"axisLabel":{"show":true,"textStyle":{"color":"rgba(0,0,0,0)","fontSize":6}}}],"series":[{"name":"患者年龄","type":"bar","barGap":20,"barWidth":15,"label":{"normal":{"show":false},"emphasis":{"show":true,"position":"left","offset":[0,0],"textStyle":{"color":"#fff","fontSize":14}}},"itemStyle":{"normal":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#6499e8"},{"offset":1,"color":"#bfddff"}]}},"emphasis":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#6499e8"},{"offset":1,"color":"#bfddff"}]}}}},{"name":"患者年龄","type":"bar","barGap":20,"barWidth":15,"xAxisIndex":2,"yAxisIndex":2,"label":{"normal":{"show":false},"emphasis":{"show":true,"position":"right","offset":[0,0],"textStyle":{"color":"#fff","fontSize":14}}},"itemStyle":{"normal":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#bfddff"},{"offset":1,"color":"#6499e8"}]}},"emphasis":{"color":{"x":1,"y":0,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#bfddff"},{"offset":1,"color":"#6499e8"}]}}}}]}', NULL, '2020-04-13 10:10:50', NULL, 4, 246, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (26, 'scatter', '排序气泡', '气泡图', 'scatter', '{"dataset":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}],"dimension":["name","value"]}', 1, '{"yAxis":[{"min":0,"gridIndex":0,"max":100,"show":false,"nameGap":30,"nameLocation":"middle"}],"xAxis":[{"min":0,"gridIndex":0,"max":100,"show":false,"nameGap":5,"nameLocation":"middle","type":"value"}],"grid":{"top":0,"bottom":0,"show":false},"series":[{"encode":{"x":"value","y":"name"},"symbol":"circle","data":[],"symbolSize":120,"itemStyle":{"normal":{"borderType":"solid","borderColor":"#fff","shadowBlur":10,"borderWidth":"4","shadowColor":"#ffc069"}},"label":{"normal":{"formatter":"{b}","color":"#fff","show":true,"textStyle":{"fontSize":"20"}}},"type":"scatter"}]}', NULL, NULL, NULL, 4, 248, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (43, 'othercomp', '科室名片', '自然资源开发利用科', 'othercomp', '{"image":"/assets/images/bk_cxbd.jpg","displayName":"自然资源开发利用科","description":"自然资源开发利用科的主要工作包括交易市场监管、节约集约利用指导、自然资源出让管理、批后监督管理等。 2020年应更加夯实国土空间管理基础，提高土地开发效益。落实《中山市国有建设用地供应管理办法》；出台闲置土地处置细则，促“闲”变“动”；量化分解各镇区年度供地任务，指导镇区分类消除供地障碍，落实“增存挂钩”机制；以“上网竞价”方式核定补缴土地出让金的市场竞争机制，提升建设用地开发利用效益。"}', 1, '{"des":{"color":"rgba(0,0,0,0.75)","fontSize":"1rem","fontWeight":"400","lineHeight":"1.5rem","paddingRight":"0.5rem"},"title":{"color":"rgba(0,0,0,0.75)","fontSize":"1.75rem","marginBottom":"1.5rem"}}', NULL, '2020-08-25 11:28:10', NULL, 4444, 1455, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (59, 'otherIcon', '相关内容', '一行图标数据', 'otherIcon', '{"code":0,"data":[{"clickUrl":"https://www.baidu.com","quotaName":"地区规划科一","icon":"icon-user","value":"23"},{"clickUrl":"https://www.baidu.com","quotaName":"地区规划科二","icon":"icon-user","value":"16"},{"clickUrl":"https://www.baidu.com","quotaName":"地区规划科三","icon":"icon-user","value":"16"}],"count":8,"message":"成功"}', 1, '{"rownameIcon":{"fontSize":"21px","marginLeft":"14px","color":"rgb(180, 190, 190)"},"rowvalueIcon":{"fontFamily":"PingFang SC","color":"rgba(0, 0, 0, 1)","fontSize":"16px","lineHeight":"46px","fontWeight":"400","marginLeft":"12px"}}', NULL, '2020-08-25 11:37:32', NULL, 6, 1419, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (6, 'index', '指标', NULL, 'index', '', NULL, NULL, NULL, NULL, NULL, NULL, NULL, 'icon-zbjc', 1, 3, 'index', NULL, NULL);
INSERT INTO graphtemplate VALUES (42, 'other', '其他', NULL, 'other', NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, 'icon-link', NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (45, 'otherpan', '总分指标', '批而未供', 'otherpan', '{"code":0,"data":[{"children":[{"unit":"宗","name":"宗数","value":"83"},{"unit":"公顷","name":"面积","value":"147.3"}],"title":"未处置"}],"message":"成功"}', 1, '{"unit":{"color":"rgba(0, 0, 0, 0.45)","fontSize":"0.75rem","lineHeight":"1.5rem","fontWeight":"400"},"name":{"color":"rgba(0, 0, 0, 0.75)","fontSize":"0.875rem","lineHeight":"1.5rem","marginTop":"0rem","fontWeight":"400"},"pic":{"color":"#fff","textAlign":"center","width":"5rem","lineHeight":"1.25rem","fontSize":"1rem","marginBottom":"0rem","height":"5rem"},"value":{"width":"100%","color":"#3c78dc","lineHeight":"2rem","fontSize":"1rem"}}', NULL, '2020-08-25 13:50:30', NULL, 6, 41160, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (1, 'piecircle', '饼图', '饼图', 'pie', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value"]}],"data":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}]}', 1, '{"color":["#90bcf5","#42CFA0","#9186CF","#D9AF80","#5BCAD4","#559BE6","#D97E87","#E6906C","#D9C57E","#BBB0C6"],"legend":{"show":false},"series":[{"encode":{"itemName":"name","seriesName":[],"tooltip":[],"value":"value"},"center":["50%","50%"],"label":{"formatter":"{b}: {d}%","fontSize":14},"labelLine":{"show":true},"type":"pie","radius":["40%","75%"]},{"data":[{"itemStyle":{"color":"rgba(250,250,250,0.3)"},"value":1}],"center":["50%","50%"],"tooltip":{"show":false},"label":{"normal":{"show":false},"emphasis":{"show":false}},"labelLine":{"normal":{"show":false},"emphasis":{"show":false}},"radius":["48%","40%"],"type":"pie","animation":false},{"hoverAnimation":false,"clockWise":false,"data":[{"name":"","itemStyle":{"normal":{"borderType":"dashed","borderColor":"rgba(90,110,130,0.1)","borderWidth":1}},"value":9}],"center":["50%","50%"],"name":"外边框","label":{"normal":{"show":false}},"type":"pie","radius":["80%","80%"]}],"tooltip":{},"title":{"left":"","text":""},"dataset":{}}', '', '2020-04-13 10:18:47', NULL, 4, 38812, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (16, 't_center_parallel', '平行四边形标题', '标题', 'titleparallel', '{"displayName":"文本内容"}', 1, '{"symboll":{"background":"linear-gradient(90deg,transparent, #98BFF5)"},"name":{"color":"rgba(0,0,0,0.75)","font-weight":"bold","fontSize":"14px","fontWeight":"700"},"symbolr":{"background":"linear-gradient(90deg, #98BFF5,transparent)"}}', NULL, '2020-10-12 13:38:10', NULL, 8, 46437, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (15, 't_left_rectangle', '双三角标题', '标题', 'titlerectangle', '{"displayName":"文本内容"}', 1, '{"color":"#3C78DC","fontSize":"0.875rem","fontWeight":"400"}', NULL, '2020-10-12 13:38:14', NULL, 8, 39525, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (28, 'txt_dqm', '双引号文本', '标题', 'text', '{"displayName":"文本内容"}', 1, '{"color":"#3C78DC","fontSize":"0.875rem","fontWeight":"400"}', NULL, '2020-02-25 16:19:29', NULL, 8, 46439, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (24, 'barh', '条形图', '条形图', 'bar', '{"dataset":[{"datadate":"2020-02-19","ljqz":189},{"datadate":"2020-02-20","ljqz":339},{"datadate":"2020-02-21","ljqz":343},{"datadate":"2020-02-22","ljqz":235},{"datadate":"2020-02-23","ljqz":212},{"datadate":"2020-02-24","ljqz":346}],"dimension":["datadate","ljqz"]}', 1, '{"yAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"splitLine":{"lineStyle":{"color":"rgba(0,0,0,0.1)"}},"axisTick":{"show":false},"axisPointer":{"show":true,"label":{"show":false}},"type":"category"},"xAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"name":"单位","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"rgba(0,0,0,0.06)"}},"type":"value","nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":12}},"barMaxWidth":"12px","legend":{"data":[]},"grid":{"x":16,"y":8,"y2":8,"x2":40,"containLabel":true},"series":[{"name":"","itemStyle":{"normal":{"color":{"x":0,"y":0,"y2":1,"x2":0,"global":false,"colorStops":[{"offset":0,"color":"#bfddff"},{"offset":1,"color":"#6499e8"}],"type":"linear"},"barBorderRadius":[100,100,100,100]}},"type":"bar"}],"tooltip":{"axisPointer":{"type":"shadow"},"trigger":"axis"},"dataset":{"source":[]}}', '', '2019-11-07 11:27:24', NULL, 4, 40464, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (33, 'gaugegrade', '分级仪表盘', '统计图表', 'gaugegrade', '{"value":67.5,"name":"风险值"}', 1, '{"tooltip":{"formatter":"{a} <br/>{b} : {c}"},"series":[{"name":"所在位置","type":"gauge","detail":{"borderColor":"#fff","shadowColor":"#fff","offsetCenter":[0,"60%"],"textStyle":{"fontWeight":"bolder"}},"axisLine":{"lineStyle":{"color":[[0.3,"#5fd24b"],[0.7,"#f5a03c"],[1,"#f05046"]]}},"axisLabel":{"fontWeight":"bolder","color":"#fff","shadowColor":"#fff","shadowBlur":0},"data":[]}]}', NULL, '2020-06-11 13:31:24', NULL, 4, 244, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (23, 't_left_pentagon', '五边形', '标题', 'titlepentagon', '{"displayName":"文本内容"}', 1, '{"color":"#3C78DC","fontSize":"0.875rem","fontWeight":"400"}', NULL, '2020-10-12 13:38:08', NULL, 8, 300, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (51, 'otherdepartment', '相关科室', '相关科室', 'otherdepartment', '[{"displayName":"自然资源用途管制科","image":"/assets/images/icon_ks@2x.png"},{"displayName":"空间规划科","image":"/assets/images/icon_ks@2x.png"}]', 1, '', NULL, '2020-08-25 14:14:15', NULL, 0, 1419, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (49, 'othertopic', '相关专题', '相关专题', 'othertopic', '[{"id":"1","displayName":"土地供应多维立体分析","clickUrl":"https://www.baidu.com","image":"/assets/images/img_tdgy.png"},{"id":"2","displayName":"城市建设用地多维立体分析","clickUrl":"https://www.baidu.com","image":"/assets/images/img_csjs.png"}]', 1, NULL, NULL, '2020-08-25 14:12:26', NULL, 0, 1421, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (58, 'list', '列表', '列表', 'list', '[{"title":"中山市过度时期控制性详细规划修改暂行规定","des":"2020-4-10按市政府要求重新报送"},{"title":"中山市过度时期控制性详细规划修改暂行规定","des":"2020-4-10按市政府要求重新报送"},{"title":"中山市过度时期控制性详细规划修改暂行规定","des":"2020-4-10按市政府要求重新报送"},{"title":"中山市过度时期控制性详细规划修改暂行规定","des":"2020-4-10按市政府要求重新报送"}]', 1, '{"listnum":{"color":"rgba(0, 0, 0, 0.75)","width":"24px","lineHeight":"22px","fontSize":"14px","fontWeight":"bold","height":"22px"},"listtitle":{"color":"rgba(0, 0, 0, 0.75)","lineHeight":"24px","fontSize":"15px","marginBottom":"4px","fontWeight":"400"},"list":{"padding":"12px 0px","margin":"0px 12px","borderBottom":"1px dotted #EBEEF5"},"listdes":{"color":"rgba(0, 0, 0, 0.45)","padding-bottom":"0px","marginBottom":"0px","lineHeight":"20px","fontSize":"13px","fontWeight":"400","height":"20px"}}', NULL, '2020-08-25 14:02:06', NULL, 0, 11199, NULL, NULL, NULL, 'text', NULL, NULL);
INSERT INTO graphtemplate VALUES (60, 'otherChoose', '模板选择器', '模板选择器', 'otherChoose', '{"classname":"choosed","children":[{"classname":"code","children":[{"label":"中山市","type":"districtId","value":"442000"},{"label":"第一分局","type":"districtId","value":"4420000101"},{"label":"第二分局","type":"districtId","value":"4420000102"},{"label":"第三分局","type":"districtId","value":"4420000103"}],"displayname":"","style":"codeOP","type":"districtId","value":"442000"},{"classname":"yearOp","children":[{"label":"否","type":"isyear","value":"0"},{"label":"是","type":"isyear","value":"1"}],"displayname":"保留年份","style":"yearOp","type":"isyear","value":"0"},{"classname":"typeOp","children":[{"label":"用地面积","type":"type","value":"1"},{"label":"建筑面积","type":"type","value":"2"},{"label":"容积率","type":"type","value":"3"}],"displayname":"","style":"typeOp","type":"type","value":"1"}],"displayname":"模板选择器"}', 1, '{"choosed":{"display": "flex"},"codeOP":{},"yearOP":{},"typeOP":{}}', NULL, '2020-12-29 10:27:08', NULL, 42, NULL, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (50, 'otherdata', '相关数据', '相关数据', 'otherdata', '[{"id":"1","displayName":"批而未供","clickUrl":"https://www.baidu.com","image":"/assets/images/icon_data@2x.png"},{"id":"2","displayName":"基准地价","clickUrl":"https://www.baidu.com","image":"/assets/images/icon_data@2x.png"},{"id":"3","displayName":"标定地价","clickUrl":"https://www.baidu.com","image":"/assets/images/icon_data@2x.png"},{"id":"4","displayName":"闲置土地","clickUrl":"https://www.baidu.com","image":"/assets/images/icon_data@2x.png"}]', 1, NULL, NULL, '2020-08-25 14:14:19', NULL, 42, 1420, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (2, 'line', '折线图', '折线图', 'line', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value"]}],"data":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}]}', 1, '{"yAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"name":"单位","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.06}},"type":"value","nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":"12px"}},"xAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"axisTick":{"show":false},"axisPointer":{"show":true,"label":{"show":false}},"type":"category","boundaryGap":false},"legend":{"top":"5%","left":"left"},"series":{"encode":{"itemName":"name","seriesName":[],"tooltip":[],"value":"code"},"areaStyle":{"color":{"x":0,"y":1,"y2":0,"x2":0,"global":false,"colorStops":[{"offset":0,"color":"rgba(66,207,160,0)"},{"offset":1,"color":"rgba(66,207,160,0.8)"}],"type":"linear"}},"color":"rgba(66,207,160,0.8)","showSymbol":false,"type":"line","smooth":0.4},"grid":{"x":16,"y":32,"y2":8,"x2":24,"containLabel":true},"tooltip":{"show":true,"trigger":"axis"},"title":{"left":"","text":""},"dataset":{"source":[],"dimensions":["",""]}}', '', '2019-11-07 11:27:24', NULL, 4, 26, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (40, 'piecircle-d', '饼图', '饼图', 'pie', '{"dataset":[{"name":"乐陵市","value":58.5},{"name":"禹城市","value":67.77},{"name":"陵城区","value":64.13},{"name":"德城区","value":55.45},{"name":"平原县","value":94.45},{"name":"临邑县","value":61.55},{"name":"齐河县","value":52.74}]}', 1, '{"backgroundColor":"#053782","color":["#6ee6a0","#f0b95f","#dcf578","#50e1d7","#f5afeb","#50b4fa","#91a0fa","#aae67d","#efafa0","#aa87dc"],"legend":{"show":false},"series":[{"encode":{"itemName":"name","seriesName":[],"tooltip":[],"value":"value"},"center":["50%","50%"],"label":{"formatter":"{b}: {d}%","fontSize":14},"labelLine":{"show":true},"type":"pie","radius":["40%","75%"]},{"data":[{"itemStyle":{"color":"rgba(250,250,250,0.3)"},"value":1}],"center":["50%","50%"],"tooltip":{"show":false},"label":{"normal":{"show":false},"emphasis":{"show":false}},"labelLine":{"normal":{"show":false},"emphasis":{"show":false}},"radius":["48%","40%"],"type":"pie","animation":false},{"hoverAnimation":false,"clockWise":false,"data":[{"name":"","itemStyle":{"normal":{"borderType":"dashed","borderColor":"rgba(255,255,255,0.2)","borderWidth":1}},"value":9}],"center":["50%","50%"],"name":"外边框","label":{"normal":{"show":false}},"type":"pie","radius":["80%","80%"]}],"tooltip":{},"title":{"left":"","text":""},"dataset":{}}', '', '2020-04-13 10:18:47', NULL, 0, 572, NULL, NULL, NULL, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (3, 'barv', '柱状图', '柱状图', 'bar', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value"]}],"data":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}]}', 1, '{"legend":{"left":"left","top":"5%"},"title":{"text":"","left":""},"tooltip":{},"dataset":{"dimensions":["",""],"source":[]},"xAxis":{"axisPointer":{"show":true,"label":{"show":false}},"splitLine":{"lineStyle":{"color":"rgba(0,0,0,0.1)"}},"axisTick":{"show":false},"axisLine":{"show":false},"axisLabel":{"color":"rgba(0,0,0,0.8)"},"type":"category"},"yAxis":{"name":"单位","nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":12},"axisLine":{"show":false},"axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"rgba(0,0,0,0.06)"}},"axisLabel":{"color":"rgba(0,0,0,0.8)"},"type":"value"},"series":{"encode":{"itemName":"","seriesName":[""],"tooltip":[""],"x":"","y":""},"type":"bar","itemStyle":{"normal":{"barBorderRadius":[100,100,0,0],"color":{"x":0,"y":0,"x2":0,"y2":1,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#bfddff"},{"offset":1,"color":"#6499e8"}]}}}},"barMaxWidth":"12px","grid":{"x":16,"y":32,"x2":16,"y2":8,"containLabel":true}}', '', '2019-11-07 11:32:18', NULL, 4, 40462, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (25, 'linevalue', '最值平均值', '最值平均值', 'line', '{"dataset":[{"name":"类目一","value":40},{"name":"类目二","value":20},{"name":"类目三","value":40},{"name":"类目四","value":35},{"name":"类目五","value":50}],"dimension":["name","value"]}', 1, '{"legend":{"left":"left","top":"5%"},"title":{"text":"","left":""},"tooltip":{"show":true,"trigger":"axis"},"dataset":{"dimensions":["",""],"source":[]},"xAxis":{"axisPointer":{"show":true,"label":{"show":false}},"axisTick":{"show":false},"axisLine":{"show":false},"axisLabel":{"color":"rgba(0,0,0,0.8)"},"type":"category","boundaryGap":false},"yAxis":{"name":"单位","nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":"12px"},"axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.06}},"axisLine":{"show":false},"axisLabel":{"color":"rgba(0,0,0,0.8)"},"type":"value"},"series":{"type":"line","markPoint":{"data":[{"type":"max","name":"最大值"},{"type":"min","name":"最小值"}]},"markLine":{"data":[{"type":"average","label":{"show":"true","position":"middle","formatter":"均值: {c}"},"name":"平均值"}],"lineStyle":{"normal":{"width":2,"color":"#9186CF"}}},"encode":{"tooltip":[],"seriesName":[],"itemName":"name","value":"code"},"color":"rgba(66,207,160,0.8)","areaStyle":{"color":{"x":0,"y":1,"x2":0,"y2":0,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"rgba(66,207,160,0)"},{"offset":1,"color":"rgba(66,207,160,0.8)"}]}},"smooth":0.4,"showSymbol":false},"grid":{"x":16,"y":44,"x2":24,"y2":8,"containLabel":true}}', '', '2019-11-07 11:27:24', NULL, 4, 38814, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (53, 'text', '纯文本', '纯文本', 'puretext', '{"displayName":"文本内容","actionurl":""}', 1, '{"fontSize":"1rem","color":"rgba(0,0,0,0.75)","fontWeight":"400","lineHeight":"1.5rem"}', NULL, '2020-09-24 14:49:09', NULL, 8, 2402, NULL, NULL, NULL, 'text', '{"mutual":[{"action":{"actionType":"newpage","contentData":{"data":{"creator":10010,"displayName":"村规-东凤镇3","creatorName":"LWJ  梁文靖","createType":1,"suffix":"jpg","type":"image/jpeg","showAddress":"https://open.chinadci.com/onemap/api/filemeta/46767","uid":"104b933946e08890c9858d18108d5c92","creatTime":"2022-04-22 17:08:50","fileSize":100112,"name":"FM967109739814125568","showType":"image","id":46767,"views":174,"md5":"141db8a3cd6e673fe41cd9738f37f1fb"},"displayName":"文件","type":"filemeta"},"content":"","contenttype":""},"source":"","event":"click","value":["default",""],"sourceNum":"default"}]}', NULL);
INSERT INTO graphtemplate VALUES (61, 'otherlist', '数据清单', '数据清单', 'otherlist', '{"code":0,"data":[{"unit":"件","name":"在办","value":"23"},{"unit":"件","name":"办结","value":"16"},{"unit":"件","name":"预警","value":"20"},{"unit":"件","name":"超期","value":"6"}],"count":4,"message":"成功"}', 1, '{"unit":{"color":"rgba(0,0,0,0.45)","textAlign":"center","fontSize":"12px"},"color":["#3c78dc","#3c78dc","#00b585","#00b585"],"name":{"color":"rgba(0,0,0,0.75)","textAlign":"center","fontSize":"14px"},"value":{"textAlign":"center","fontSize":"18px","fontWeight":"bold"},"listback":{}}', NULL, '2021-01-18 11:06:01', NULL, 6, 41151, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (71, 'otherdynamic', '每周动态', '每周动态', 'otherdynamic', '{"displayName":"","description":"局内重点工作推进情况、工作动态等","time":"https://fzjc.apps.chinadci.com/zs/api/filemeta/12321?isdownfile=0","leftpic":"https://fzjc.apps.chinadci.com/zs/api/filemeta/12320?isdownfile=0","rightpic":"https://fzjc.apps.chinadci.com/zs/api/filemeta/12310?isdownfile=0"}', 1, '{"desp":{"color":"rgba(0, 0, 0, 0.45)","fontSize":"12px","paddingTop":"8px","fontWeight":200},"name":{},"rightpic":{"margin":"8px 0px 0px ","width":"84px","height":"18px"},"dynamic":{"box-shadow":"0px 2px 0px 0px rgba(32, 159, 132, 0.2)","borderRadius":"6px","alignItems":"center","background":"#F0FAF8","display":"flex","position":"relative","height":"72px"},"time":{"alignContent":"center","color":"#FFFFFF","top":"-4px","display":"flex","width":"80px","fontSize":"13px","lineHeight":"20px","position":"absolute","right":"10px","fontWeight":400,"justifyContent":"center","height":"20px"},"actionurl":"/layout/dynamic/sjdt?title=市局每周动态","leftpic":{"marginRight":"15px","width":"75px","height":"44px"}}', NULL, '2021-03-25 14:56:20', NULL, 4444, 12310, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (65, 'otherrepro', '进度条组', '进度条组', 'otherrepro', '{"metadata":[{"displayname":"测试数据","fields":[{"fieldname":"面积","fieldkey":"area"},{"fieldname":"percent","fieldkey":"百分比"},{"fieldname":"displayName","fieldkey":"指标名称"}],"dimension":["area","percent"]}],"code":0,"data":[{"children":[{"unit":"公顷","name":"面积","value":35,"key":"area"},{"unit":"%","name":"百分比","value":86,"key":"percent"}],"displayName":"已用"},{"children":[{"unit":"公顷","name":"面积","value":240,"key":"area"},{"unit":"%","name":"百分比","value":24,"key":"percent"}],"displayName":"剩余"}],"count":2,"message":""}', 1, '{"unit":{"marginRight":"7px","whiteSpace":"nowrap","color":"rgba(0,0,0,0.45)","fontSize":"12px","fontWeight":"400"},"progdom":{"marginRight":"16px","color":["#879EAA","#209F84"],"display":"flex","width":"70px","height":"12"},"progressField":"percent","name":{"marginRight":"18px","whiteSpace":"nowrap","color":"rgba(0,0,0,0.75)","fontSize":"12px","fontWeight":"400"},"unit1":{"color":"rgba(0,0,0,0.45)","fontSize":"12px","fontWeight":"400"},"row":{"alignItems":"center","display":"flex","marginBottom":"13px"},"value":{"marginRight":"2px","color":"rgba(0,0,0,0.75)","fontSize":"14px","fontWeight":"Bold","marginLeft":"7px"}}', NULL, '2021-01-21 10:31:25', NULL, 6, 41162, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (64, 'otherdate', '日期表', '日期表', 'otherdate', '{year: 2020, month: 12,unit:"月"}', 1, '{"container":{"padding":"0px","margin":"0px"},"monthUnit":{"fontFamily":"PingFang SC","color":"rgba(0, 0, 0, 0.45)","fontSize":"12px","fontWeight":400},"wrapp":{"borderRadius":"4px","alignItems":"center","background":"rgba(32, 159, 132, 0.9)","flexDirection":"column","display":"flex","width":"52px","height":"58px"},"month":{"fontFamily":"PingFang SC","borderRadius":"4px","color":"rgba(0, 0, 0, 0.75)","textAlign":"center","background":"#FFFFFF","width":"44px","lineHeight":"34px","fontSize":"15px","fontWeight":"bold","height":"34px"},"year":{"fontFamily":"PingFang SC","color":"#FFFFFF","display":"flex","fontSize":"12px","justifyContent":"spaceAround","fontWeight":400},"yearLeftDot":{"marginRight":"4px"},"yearRightDot":{"marginLeft":"4px"}}', NULL, '2021-01-20 18:10:36', NULL, 42, 11241, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (82, 'multipleLinkage', '多图联动', '多图联动', 'multipleLinkage', '{"code":0,"message":"","count":1,"metadata":[{"displayname":"","fields":[{"fieldkey":"receive","fieldname":"A","unit":"件"},{"fieldkey":"deal","fieldname":"B","unit":"件"},{"fieldkey":"completed","fieldname":"C","unit":"件"},{"fieldkey":"overdue","fieldname":"D","unit":"件"},{"fieldkey":"datetime","fieldname":"时间"},{"fieldkey":"caseUnit","fieldname":"单位"},{"fieldkey":"districtName","fieldname":"机构名称"}],"dimension":["datetime","receive","deal","completed","overdue"]}],"data":[{"caseUnit":"件","receive":137,"datetime":"2015","deal":93,"overdue":5,"completed":14},{"caseUnit":"件","receive":156,"datetime":"2016","deal":96,"overdue":0,"completed":11},{"caseUnit":"件","receive":146,"datetime":"2017","deal":107,"overdue":4,"completed":14},{"caseUnit":"件","receive":155,"datetime":"2018","deal":91,"overdue":1,"completed":11},{"caseUnit":"件","receive":137,"datetime":"2019","deal":95,"overdue":0,"completed":11},{"caseUnit":"件","receive":159,"datetime":"2020","deal":104,"overdue":0,"completed":15},{"caseUnit":"件","receive":123,"datetime":"2021","deal":75,"overdue":0,"completed":7}]}', 1, '{"yAxis":[{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"min":0,"gridIndex":0,"axisLine":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.1}},"axisTick":{"show":false},"name":"","axisPointer":{"show":false,"label":{"show":false}},"splitNumber":4,"type":"value"}],"xAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.06}},"type":"category","boundaryGap":[0,0.01],"nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":"12px"}},"color":["#209F84","#28B2BF","#879EAA","#FBBE54"],"legend":{"itemGap":16,"data":["收件","受理","办结","超期"],"itemHeight":10,"show":false,"icon":"circle","itemWidth":10},"grid":{"top":"55%"},"series":[{"type":"line","seriesLayoutBy":"row","smooth":true},{"type":"line","seriesLayoutBy":"row","smooth":true},{"type":"line","seriesLayoutBy":"row","smooth":true},{"type":"line","seriesLayoutBy":"row","smooth":true},{"encode":{"itemName":"field","value":[1]},"center":["50%","25%"],"emphasis":{"focus":"data"},"id":"pie","label":{"formatter":"{b}: {@[1]} ({d}%)"},"type":"pie","radius":"30%"}],"tooltip":{"axisPointer":{"type":"shadow"},"trigger":"axis","confine":"true"},"title":{"subtext":"","text":""},"dataset":{"source":[{"caseUnit":"件","receive":137,"datetime":"2015","deal":93,"overdue":5,"completed":14},{"caseUnit":"件","receive":156,"datetime":"2016","deal":96,"overdue":0,"completed":11},{"caseUnit":"件","receive":146,"datetime":"2017","deal":107,"overdue":4,"completed":14},{"caseUnit":"件","receive":155,"datetime":"2018","deal":91,"overdue":1,"completed":11},{"caseUnit":"件","receive":137,"datetime":"2019","deal":95,"overdue":0,"completed":11},{"caseUnit":"件","receive":159,"datetime":"2020","deal":104,"overdue":0,"completed":15},{"caseUnit":"件","receive":123,"datetime":"2021","deal":75,"overdue":0,"completed":7}],"dimensions":["datetime","receive","deal","completed","overdue"]}}', NULL, '2021-07-27 17:14:36', NULL, 4, 36487, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (39, 'barh-d', '条形图', '条形图', 'bar', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value"]}],"data":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}]}', 1, '{"yAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.1}},"axisTick":{"show":false},"axisPointer":{"show":true,"label":{"show":false}},"type":"category"},"xAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)"},"axisLine":{"show":false},"name":"万人","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.06}},"type":"value","nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":"12px"}},"barMaxWidth":"12px","legend":{"data":[]},"grid":{"x":16,"y":8,"y2":8,"x2":40,"containLabel":true},"series":[{"name":"2011年","itemStyle":{"normal":{"color":{"x":0,"y":0,"y2":1,"x2":0,"global":false,"colorStops":[{"offset":0,"color":"#bfddff"},{"offset":1,"color":"#6499e8"}],"type":"linear"},"barBorderRadius":[100,100,100,100]}},"type":"bar"}],"tooltip":{"axisPointer":{"type":"shadow"},"trigger":"axis"},"dataset":{"source":[]}}', '', '2019-11-07 11:27:24', NULL, 0, 574, NULL, NULL, NULL, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (54, 'othertime', '时间轴', '时间轴', 'othertime', '[{"content":"活动开始","timestamp":"2018-04-15","status":"完成"},{"type":"type=1","value":"type=2","content":"通过审核","timestamp":"2018-04-13","status":"进行中"},{"type":"type=1","value":"type=2","content":"创建成功","timestamp":"2018-04-11","status":"未开始"},{"type":"type=1","value":"type=2","content":"创建成功1","timestamp":"2018-04-11","status":"未开始"},{"type":"type=1","value":"type=2","content":"创建成功2","timestamp":"2018-04-11","status":"未开始"},{"type":"type=1","value":"type=2","content":"创建成功3","timestamp":"2018-04-11","status":"未开始"}]', 1, '{"finshCrile":{"border":"2px solid transparent","borderColor":"#209F84","backgroundColor":"#fff","width":"5px","height":"5px"},"comingCrile":{"padding":"1px","borderColor":"#209F84","backgroundColor":"#fff","children":{"backgroundColor":"#209F84","width":"5px","height":"5px"},"width":"7px","height":"8px"},"finsh":{"color":"rgba(0, 0, 0, 1)","fontSize":"12px","fontWeight":"600"},"normalCrile":{"border":"2px solid transparent","backgroundColor":"#fff","borderColor":"rgba(0, 0, 0, 0.45)","width":"5px","height":"5px"},"desc":{"color":"rgba(0, 0, 0, 0.45)","fontSize":"12px","fontWeight":"400"}}', NULL, '2020-08-26 21:02:47', NULL, 4444, 11245, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (57, 'otherlineindex', '一行指标数据', '一行指标数据', 'otherlineindex', '{"code":0,"data":[{"unit":"亿元","name":"成交金额","value":"23"},{"unit":"%","name":"同比","value":"16"}],"count":8,"message":"成功"}', 1, '{"value":{"marginRight":"0.375rem","fontSize":"1.125rem","lineHeight":"48px"},"name":{"color":"rgba(0, 0, 0, 0.75)","fontSize":"0.875rem","lineHeight":"48px"},"unit":{"color":"rgba(0, 0, 0, 0.45)","fontSize":"0.75rem","fontWeight":"400","height":"48px"},"row":{"borderLeft":0,"borderRadius":"10px","borderRight":0,"lineHeight":"48px","height":"48px"}}', NULL, '2020-08-25 11:37:32', NULL, 6, 41158, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (20, 'barhmult', '多条形图', '多条形图', 'barhmulti', '{"dataset":[{"datadate":"2020-02-16","ljqz":339,"ljys":112},{"datadate":"2020-02-17","ljqz":339,"ljys":3},{"datadate":"2020-02-18","ljqz":339,"ljys":0},{"datadate":"2020-02-19","ljqz":339,"ljys":20},{"datadate":"2020-02-20","ljqz":339,"ljys":10},{"datadate":"2020-02-21","ljqz":343,"ljys":43},{"datadate":"2020-02-22","ljqz":345,"ljys":280},{"datadate":"2020-02-23","ljqz":345,"ljys":210},{"datadate":"2020-02-24","ljqz":346,"ljys":0}],"dimension":["datadate","ljqz","ljys"]}', 1, '{"color":["#90bcf5","#42CFA0","#9186CF","#D9AF80","#5BCAD4","#559BE6","#D97E87","#E6906C","#D9C57E","#BBB0C6"],"title":{"text":"","subtext":""},"tooltip":{"trigger":"axis","axisPointer":{"type":"shadow"}},"dataset":{"dimensions":[],"source":[]},"legend":{"data":["2011年","2012年"]},"grid":{"x":16,"y":32,"x2":16,"y2":16,"containLabel":true},"xAxis":{"nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":"12px"},"axisLine":{"show":false},"axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#000","opacity":0.06}},"axisLabel":{"color":"rgba(0,0,0,0.8)"},"type":"value","boundaryGap":[0,0.01]},"yAxis":{"axisPointer":{"show":true,"label":{"show":false}},"splitLine":{"lineStyle":{"color":"#000","opacity":0.1}},"axisTick":{"show":false},"axisLine":{"show":false},"axisLabel":{"color":"rgba(0,0,0,0.8)"},"type":"category"},"series":[{"name":"2011年","type":"bar"},{"name":"2012年","type":"bar"}]}', '', '2019-11-07 11:27:24', NULL, 4, 38810, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (48, 'titleleftline', '左划线标题', '左划线标题', 'titleleftline', '{"code":0,"data":[{"updateDate":"更新于2020-9-18","displayname":"公开出让"}],"message":"成功"}', 1, '{"iconstyle":{"color":"rgba(0,0,0,0.35)","fontSize":"0.875rem","height":"20px"},"datadesp":"","more":"icon-arrowright","line":{"padding":"28px 0px 16px 0px","backgroundColor":"#fff"},"icon":"icon-question-c","datadespurl":"","pic":{"backgroundColor":"#209F84","width":"0.25rem","height":"1rem"},"actionurl":"","morestyle":{"marginRight":"0.625rem","color":"rgba(0,0,0,0.45)","fontSize":"1rem","fontWeight":"400","marginLeft":"auto"},"sourceUrl":"","datadespstyle":{},"name":{"marginRight":"0.5rem","whiteSpace":"nowrap","color":"rgba(0, 0, 0, 0.75)","fontSize":"1.0625rem","fontWeight":"bold"},"time":{"marginRight":"0.25rem","color":"rgba(0, 0, 0, 0.45)","fontSize":"0.6875rem","fontWeight":"400"}}', NULL, '2020-08-25 14:02:06', NULL, 8, 46438, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (38, 'barv-d', '柱状图', '柱状图', 'bar', '{"code":0,"message":"","count":5,"metadata":[{"fields":[{"fieldkey":"name","fieldname":"名称"},{"fieldkey":"value","fieldname":"数值"}],"dimension":["name","value"]}],"data":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}]}', 1, '{"legend":{"left":"left","top":"5%"},"title":{"text":"","left":""},"tooltip":{},"dataset":{"dimensions":["qw","zxss"],"source":[]},"xAxis":{"axisPointer":{"show":true,"label":{"show":false}},"splitLine":{"lineStyle":{"color":"#fff","opacity":0.2}},"axisTick":{"show":false},"axisLine":{"show":false},"axisLabel":{"color":"rgba(255,255,255,0.8)"},"type":"category"},"yAxis":{"name":"","nameTextStyle":{"color":"rgba(255,255,255,0.5)","fontSize":"12px"},"axisLine":{"show":false},"axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#fff","opacity":0.1}},"axisLabel":{"color":"rgba(255,255,255,0.8)"},"type":"value"},"series":{"encode":{"itemName":"qw","seriesName":[""],"tooltip":["zxss"],"x":"qw","y":"zxss"},"type":"bar","itemStyle":{"normal":{"barBorderRadius":[100,100,0,0],"color":{"x":0,"y":0,"x2":0,"y2":1,"type":"linear","global":false,"colorStops":[{"offset":0,"color":"#50dcff"},{"offset":1,"color":"#3c8cf0"}]}}}},"barMaxWidth":"12px","grid":{"x":16,"y":20,"x2":16,"y2":8,"containLabel":true}}', '', '2019-11-07 11:32:18', NULL, 0, 575, NULL, NULL, NULL, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (46, 'tablefz', '分组表格', '分组表格', 'tablefz', '{"code":0,"message":"","count":4,"metadata":[{"name":"quota","displayName":"指标","children":[]},{"name":"thisMouth","displayName":"本月","children":[]},{"name":"lastYearMouth","displayName":"去年同期","children":[]},{"name":"compare","displayName":"同比","children":[]}],"style":null,"data":[{"quota":"总面积","thisMouth":"35","lastYearMouth":"33","unit":null,"compare":"+2%"},{"quota":"宗数","thisMouth":"4","lastYearMouth":"4","unit":null,"compare":"0%"},{"quota":"成交额","thisMouth":"2342","lastYearMouth":"2300","unit":null,"compare":"+2%"},{"quota":"成交均价","thisMouth":"2314","lastYearMouth":"2456","unit":null,"compare":"+14%"}]}', 1, '{"border":false,"header1":{"margin":16,"backgroundColor":"rgba(233, 245, 242,1)","color":"rgba(0,0,0,0.75)","textAlign":"center","height":"40px"},"body1":{"backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","fontWeight":400,"height":"40px"},"body2":{"backgroundColor":"#fafafa","color":"rgba(0,0,0,0.75)","fontWeight":400,"height":"40px"},"header":{"margin":16,"backgroundColor":"#e8f4ff","color":"rgba(0,0,0,0.75)","textAlign":"center","fontWeight":600,"height":"40px"},"cellstyle":{"textAlign":"center","width":120,"borderBottom":"1px solid #EBEEF5"},"tablestyle":{"backgroundColor":"#fff"}}', NULL, '2020-08-25 13:54:50', NULL, 11, 1423, NULL, NULL, NULL, 'table', NULL, 1);
INSERT INTO graphtemplate VALUES (70, 'barthstack', '堆叠图', '堆叠图', 'barthstack', '{"dataset":[{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"工矿仓储","mj":0,"zfcount":100},{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"商服","mj":0,"zfcount":200},{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"住宅","mj":0,"zfcount":300},{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"其他","mj":0,"zfcount":400}],"dimensions":["label","zfcount","zscount","zs","zfarea","zsarea","mj"]}', 1, '{"yAxis":[{"axisLabel":{"show":true},"inverse":true,"min":0,"max":400,"minInterval":0,"axisLine":{"show":false},"name":"宗","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#e6e6e6"},"show":true},"type":"value","nameTextStyle":{"color":"rgba(0,0,0,0.45)"}},{"gridIndex":1,"show":true},{"axisLabel":{"show":true},"min":0,"max":400,"gridIndex":2,"minInterval":100,"axisLine":{"show":false},"name":"公顷","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"#e6e6e6"},"show":true},"type":"value","nameTextStyle":{"color":"rgba(0,0,0,0.45)"}}],"xAxis":[{"axisLabel":{"show":false},"inverse":false,"axisLine":{"lineStyle":{"color":"#e6e6e6"}},"axisTick":{"show":false},"position":"right","type":"category"},{"axisLabel":{"padding":[8,0,0,0],"color":"#262C41","fontSize":12,"align":"center"},"gridIndex":1,"axisLine":{"show":false},"position":"","type":"category"},{"axisLabel":{"margin":20,"color":"#000","show":true},"gridIndex":2,"axisLine":{"lineStyle":{"color":"#E0E0E0"}},"axisTick":{"show":false},"position":"left","type":"category"}],"legend":{"data":["已处置","未处置"],"itemHeight":10,"icon":"circle"},"grid":[{"top":"58%","left":"16px","right":"16px","height":"32%","containLabel":true},{"top":"40%","height":"0%"},{"top":"12%","left":"16px","right":"16px","height":"40%","containLabel":true}],"series":[{"barWidth":22,"stack":"one","name":"已处置","itemStyle":{"color":"#209F84"},"label":{"color":"#fff00","show":false},"type":"bar"},{"encode":{"itemName":"label","seriesName":["未处置222"],"tooltip":[3,4],"value":"zfcount"},"barWidth":22,"stack":"one","name":"未处置1","itemStyle":{"color":"#28B2BF"},"label":{"color":"#fff00","show":false},"type":"bar"},{"barWidth":22,"barGap":"-100%","name":"宗数","itemStyle":{"color":"#ffffff00"},"label":{"color":"#000","show":false,"position":"bottom"},"type":"bar"},{"barWidth":22,"stack":"two","xAxisIndex":2,"name":"政府原因","itemStyle":{"color":"#209F84"},"label":{"color":"#fff00","show":false},"type":"bar","yAxisIndex":2},{"barWidth":22,"stack":"two","xAxisIndex":2,"name":"自身原因","itemStyle":{"color":"#28B2BF"},"label":{"color":"#fff00","show":false},"type":"bar","yAxisIndex":2},{"barWidth":22,"barGap":"-100%","xAxisIndex":2,"name":"面积","itemStyle":{"color":"#ffffff00"},"label":{"color":"#000","show":false,"position":"top"},"type":"bar","yAxisIndex":2}],"tooltip":{},"dataset":{"source":[{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"工矿仓储","mj":0,"zfcount":100},{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"商服","mj":0,"zfcount":200},{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"住宅","mj":0,"zfcount":300},{"zfarea":0,"zsarea":0,"zs":0,"zscount":0,"label":"其他","mj":0,"zfcount":400}],"dimensions":["label","zfcount","zscount","zs","zfarea","zsarea","mj"]}}', '', '2019-11-07 11:27:24', NULL, 4, 38128, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (76, 'attachedPic', '图片预览', '图片预览', 'attachedPic', '{"imageUrl":"https://open.chinadci.com/onemap/api/filemeta/28686?resolutionratio=original","name":"预览例图"}', NULL, '{"icon":{"fontSize":"1rem","lineHeight":"2.25rem"},"label":{"margin":"0 0.625rem 0 0.375rem","backgroundColor":"#209F84","borderRadius":"1rem","width":"0.25rem","height":"0.25rem"},"title":{"marginRight":"0.25rem","fontSize":"0.875rem"},"type":"icon"}', NULL, '2021-06-30 18:25:59', NULL, 42, NULL, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (83, 'pictorialBar', '画报', '画报', 'pictorialBar', '{"code":0,"message":"","count":1,"metadata":[{"displayname":"","fields":[],"dimension":["name","value","unit"]}],"data":[{"name":"地上停车场","value":5341.64,"unit":"万个"},{"name":"地下停车场","value":12284.67,"unit":"万个"}]}', 1, '{"tooltip":{},"xAxis":{"show":false},"yAxis":{"data":["",""],"inverse":true,"axisTick":{"show":false},"axisLine":{"show":false}},"grid":{"top":"center","height":200,"left":70,"right":100},"series":[{"type":"pictorialBar","symbol":"image://data:image/png;base64,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","symbolRepeat":"fixed","symbolMargin":"5%","symbolClip":true,"symbolSize":30,"data":[41.912,19.1]},{"type":"pictorialBar","itemStyle":{"opacity":0.2},"animationDuration":0,"symbolRepeat":"fixed","symbolMargin":"5%","symbol":"image://data:image/png;base64,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","symbolSize":30,"data":[80,80],"z":5}]}', NULL, '2021-10-09 11:23:23', NULL, 4444, 38480, NULL, NULL, NULL, 'graph', NULL, NULL);
INSERT INTO graphtemplate VALUES (81, 'stairs', '阶梯瀑布图', '阶梯瀑布图', 'stairs', '{
	"metadata": [
		{
			"dataIndex": "month",
			"title": "月份",
			"key": "month"
		},
		{
			"dataIndex": "total",
			"title": "总量",
			"key": "total"
		},
		{
			"dataIndex": "reality",
			"title": "收入",
			"key": "income"
		},
		{
			"dataIndex": "predict",
			"title": "支出",
			"key": "expense"
		}
	],
	"data": [
		[
			1,
			2,
			3,
			4,
			5,
			6,
			7,
			8,
			9,
			10,
			11,
			12
		],
		[
			0,
			900,
			1245,
			1530,
			1376,
			1376,
			1511,
			1689,
			1856,
			1495,
			1292
		],
		[
			900,
			345,
			393,
			"-",
			"-",
			135,
			178,
			286,
			"-",
			"-",
			"-"
		],
		[
			"-",
			"-",
			"-",
			108,
			154,
			"-",
			"-",
			"-",
			119,
			361,
			203
		]
	],
	"style": {
		"dimensions": [
			"month",
			"total",
			"income",
			"expense"
		]
	}
}', 1, '{"tooltip":{"trigger":"axis","axisPointer":{"type":"shadow"}},"legend":{"data":["支出","收入"]},"grid":{"left":"3%","right":"4%","bottom":"3%","containLabel":true},"xAxis":{"type":"category","splitLine":{"show":false}},"yAxis":{"type":"value"},"series":[{"name":"辅助","type":"bar","stack":"总量","itemStyle":{"barBorderColor":"rgba(0,0,0,0)","color":"rgba(0,0,0,0)"},"emphasis":{"itemStyle":{"barBorderColor":"rgba(0,0,0,0)","color":"rgba(0,0,0,0)"}},"seriesLayoutBy":"row"},{"name":"收入","type":"bar","stack":"总量","label":{"show":true,"position":"top"},"seriesLayoutBy":"row"},{"name":"支出","type":"bar","stack":"总量","label":{"show":true,"position":"bottom"},"seriesLayoutBy":"row"}],"dataset":{}}', NULL, '2021-07-27 17:14:36', NULL, 4, 38946, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (72, 'textdynamic', '动态文本', '动态文本', 'textdynamic', '{"text1":1521,"text2":205}', 1, '{
    "default": "重大平台内存量用地，其中储备用地总规模约<span style={a=}>{text1=1521}</span>亩，共<span style={a=}>{text2=1521}</span>宗；批而未供总规模约<span style={b=}>{text3=1521}</span>亩，共<span style={b=}>{text4=1521}</span>宗；供而未建总规模约<span style={b=}>{text5=1521}</span>亩，共<span style={b=}>{text6=1521}</span>宗。",
    "a": {
        "color": "#3c78dc"
    },
    "b": {
        "color": "#00b585"
    },
    "defstyle": {
        "padding": "12px",
        "color": "rgba(0, 0, 0, 0.75)",
        "borderRadius": "8px",
        "alignItems": "center",
        "display": "flex",
        "width": "100%",
        "fontSize": "14px",
        "lineHeight": "1.5rem",
        "textIndent": "26px",
        "fontWeight": "400",
        "justifyContent": "center"
    }
}', NULL, '2021-03-25 15:07:27', NULL, 8, 39526, NULL, NULL, NULL, 'text', NULL, 1);
INSERT INTO graphtemplate VALUES (78, 'currency', '通用组件', '通用组件', 'currency', '{"data":[{"classname":"code","level":"province","children":[{"name":"中山市","type":"districtId","value":"442000"},{"name":"第一分局","type":"districtId","value":"4420000101"},{"name":"第二分局","type":"districtId","value":"4420000102"},{"name":"第三分局","type":"districtId","value":"4420000103"}],"name":"","style":"codeOP","type":"districtId","value":"442000"}]}', 1, '{"type":"commonChoose"}', NULL, '2021-07-09 16:09:27', NULL, 42, 39194, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (52, 'othertab', '复合参数', '复合标签', 'othertab', '{"child":[{"dispalyname":"标签一","type":"btntab","children":[{"displayname":"用地面积","label":"ydmj","type":"type=1","value":"type=2"},{"displayname":"建筑面积","label":"jzmj","type":"type=1","value":"type=2"},{"displayname":"平均容积率","label":"pjrjl","type":"type=1","value":"type=3"}]},{"dispalyname":"下拉框","type":"option","children":[{"label":"全部","type":"type","value":"all"},{"label":"已竣工","type":"type","value":"yjg"},{"label":"已动工未竣工","type":"type","value":"ydgwjg"},{"label":"未动工","type":"type","value":"wdg"}]},{"dispalyname":"tab","type":"blinetab","children":[{"displayname":"图表","label":"graph"}]}]}', 1, '{"optionstyle":{"display":"none"},"btnstyle":{"default":{"border":"none","padding":"0px 12px","margin":"16px 2px","backgroundColor":"#fff","color":"rgba(32,159,132,1)","borderRadius":"1rem","fontSize":"0.875rem","lineHeight":"2rem","height":"2rem"},"primary":{"border":"none","padding":"0px 12px","margin":"0px 2px","backgroundColor":"rgba(238,247,245,1)","color":"rgba(32,159,132,1)","borderRadius":"1rem","fontSize":"0.875rem","lineHeight":"2rem","fontWeight":"700","height":"2rem"}},"tabstyle":{"default":{"border":"none","margin":"8px","backgroundColor":"#fff","color":"rgba(0,0,0,0.45)","borderRadius":"0rem","fontSize":"0.875rem","lineHeight":"2rem","height":"2rem"},"primary":{"borderLeft":"1px","margin":"8px","backgroundColor":"rgba(255,255,255,1)","color":"rgba(32,159,132,1)","borderRadius":"0rem","borderRight":"1px","fontSize":"0.875rem","lineHeight":"2rem","borderTop":"1px","borderBottom":"solid 2px ","fontWeight":"700","height":"2rem"}}}', NULL, '2020-08-26 21:02:47', NULL, NULL, 2488, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (79, 'nestpie', '嵌套圆环图', '嵌套圆环图', 'nestpie', '{"dataset":[{"name":"数据一","value":1},{"name":"数据二","value":2},{"name":"数据三","value":3},{"name":"数据四","value":4},{"name":"数据五","value":5}],"dimensions":["name","value"]}', 1, '{"color":["#4178dc","#f07d55","#98b8dc","#1abeb9","#8f9eb3","#f9c36a","#98b8dc","#8f9eb3","#D97E87","#E6906C","#D9C57E","#BBB0C6"],"legend":{"itemHeight":10,"show":true,"icon":"circle","itemWidth":10},"series":[{"top":25,"groupIndexs":[3,4,5],"selectedMode":"single","name":"访问来源","label":{"show":false,"fontSize":14,"position":"inner"},"labelLine":{"show":false},"type":"pie","radius":[0,"30%"]},{"top":25,"groupIndexs":[1,2,3,5],"name":"访问来源","labelLine":{"length":30},"label":{"formatter":"{a|{a}}{abg|}\n{hr|}\n  {b|{b}：}{@value}  {per|{d}%}  ","backgroundColor":"#F6F8FC","borderColor":"#8C8D8E","borderRadius":4,"borderWidth":1,"rich":{"a":{"color":"#6E7079","lineHeight":22,"align":"center"},"b":{"color":"#4C5058","fontSize":14,"lineHeight":33,"fontWeight":"bold"},"hr":{"borderColor":"#8C8D8E","borderWidth":1,"width":"100%","height":0},"per":{"padding":[3,4],"backgroundColor":"#4C5058","color":"#fff","borderRadius":4}}},"type":"pie","radius":["45%","60%"]}],"grid":{"x":6,"y":32,"y2":16,"x2":6},"tooltip":{"trigger":"item"},"dataset":{"source":[{"name":"数据一","value":1},{"name":"数据二","value":2},{"name":"数据三","value":3},{"name":"数据四","value":4},{"name":"数据五","value":5}],"dimensions":["name","value"]}}', NULL, '2021-07-13 16:09:27', NULL, 4, 31539, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (47, 'echartmap', '地图', '地图', 'echartmap', '[{"displayname":"板芙镇","value":40.38},{"displayname":"大涌镇","value":52.73},{"displayname":"东凤镇","value":36.56},{"displayname":"东区","value":27.23},{"displayname":"东升镇","value":133},{"displayname":"阜沙镇","value":89.43},{"displayname":"港口镇","value":168.36},{"displayname":"古镇镇","value":52.73},{"displayname":"横栏镇","value":36.56},{"displayname":"黄圃镇","value":27.23},{"displayname":"火炬区","value":133},{"displayname":"民众镇","value":89.43},{"displayname":"翠亨新区起步区","value":168.36},{"displayname":"南朗镇","value":89.43},{"displayname":"南区","value":168.36},{"displayname":"南头镇","value":52.73},{"displayname":"三角镇","value":36.56},{"displayname":"三乡镇","value":27.23},{"displayname":"神湾镇","value":133},{"displayname":"石岐区","value":89.43},{"displayname":"坦洲镇","value":168.36},{"displayname":"五桂山","value":27.23},{"displayname":"西区","value":133},{"displayname":"小榄镇","value":89.43},{"displayname":"沙溪镇","value":168.36}]', 1, '{"code":442000,"series":[{"data":[{"name":"zs","label":{"normal":{"show":false}}}],"name":"","map":"zs","mapType":"","itemStyle":{"normal":{"borderColor":"#e6dcd2","areaColor":"transparent","shadowBlur":24,"borderWidth":1.5,"shadowColor":"rgba(250, 185, 185, 0.3)"},"emphasis":{"areaColor":"#bfddff"}},"type":"map","roam":false}],"tooltip":{"formatter":"{b}:{c}","trigger":"item"},"toolbox":{"top":"top","feature":{"saveAsImage":{},"restore":{},"dataView":{"readOnly":false}},"left":"right","show":false},"visualMap":{"min":0,"max":100,"left":"16","calculable":true,"bottom":"16","itemHeight":144,"show":false,"itemWidth":24,"text":["600","0"],"textStyle":{"color":"#333","fontSize":12},"inRange":{"color":["#FAF0DC","#FAC8A5","#F08C69","#D7554B"]},"showLabel":true}}', NULL, '2020-08-25 13:58:44', NULL, 4, 38816, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (80, '桑基图', '桑基图', '桑基图', 'sankey', '{"code":0,"message":"","count":1,"metadata":[{"displayname":"","fields":[],"dimension":[]}],"data":[{"source":"Werne","target":"Duesseldorf","value":357.83},{"source":"Duesseldorf","target":"Cambridge","value":894.59},{"source":"Werne","target":"Cambridge","value":178.91}]}', 1, '{"title":{"subtext":"","left":"center"},"colors":["#f18bbf","#0078D7","#3891A7","#0037DA","#C0BEAF","#EA005E","#D13438","#567C73","#9ed566","#2BCC7F","#809B48","#9B2D1F"],"series":[{"type":"sankey","left":50,"top":20,"right":150,"bottom":25,"data":[],"links":[],"lineStyle":{"color":"source","curveness":0.5},"itemStyle":{"color":"#1f77b4","borderColor":"#1f77b4"},"label":{"color":"rgba(0,0,0,0.7)","fontFamily":"Arial","fontSize":10}}],"tooltip":{"trigger":"item"}}', NULL, '2021-07-26 16:58:12', NULL, 4, 38811, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (85, 'iframe', 'iframe标签', 'iframe标签', 'otheriframe', '{"data":[{"url":"https://apps.chinadci.com/onemapweb/views/topicview/908660172510388224?title=耕地占用_批准用地占用耕地类型"}]}', 1, '{"frameStyle":{"width":"100%","height":"100%"},"type":"comIframe"}', NULL, '2021-12-15 10:37:10', NULL, 42, 39904, NULL, NULL, NULL, 'other', NULL, NULL);
INSERT INTO graphtemplate VALUES (21, 'gauge', '仪表盘', '仪表盘', 'gauge', '{"value":25,"name":"完成80%以上"}', 1, '{"series":[{"pointer":{"show":false},"startAngle":180,"data":[{"name":"","value":80}],"max":100,"center":["50%","70%"],"endAngle":0,"splitNumber":0,"type":"gauge","axisLabel":{"color":"transparent"},"min":0,"axisLine":{"lineStyle":{"opacity":0}},"name":"白色圈刻度","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"transparent","width":2},"length":16},"z":4,"detail":{"show":false},"radius":"85%"},{"pointer":{"show":false},"startAngle":180,"data":[{"show":false,"value":" "}],"center":["50%","70%"],"endAngle":0,"type":"gauge","axisLabel":{"show":false},"axisLine":{"lineStyle":{"color":[[0,"#fff"],[1,"#bfddff"]],"shadowBlur":0,"width":24}},"name":"灰色内圈","splitLine":{"show":false},"axisTick":{"show":false},"z":2,"detail":{"show":0},"radius":"110%"},{"pointer":{"show":true,"width":5,"length":"55%"},"startAngle":180,"data":[],"center":["50%","70%"],"endAngle":0,"type":"gauge","axisLabel":{"show":false},"axisLine":{"lineStyle":{"color":[[1,"#3c78dc"]],"width":0}},"name":"指针","axisTick":{"show":false},"splitLine":{"show":false},"z":6,"detail":{"show":0},"radius":"95%"},{"pointer":{"show":0},"startAngle":180,"max":100,"center":["50%","70%"],"endAngle":0,"splitNumber":5,"type":"gauge","axisLabel":{"distance":4,"textStyle":{"color":"#071542","fontSize":"12","fontWeight":"bold"}},"min":0,"axisLine":{"lineStyle":{"color":[[1,"#6499e8"]],"width":3},"show":true},"splitLine":{"lineStyle":{"color":"#6499e8"},"show":true,"length":12},"axisTick":{"lineStyle":{"color":"#6499e8","width":1},"show":true,"length":8,"splitNumber":8},"detail":{"show":0},"radius":"88%"}],"tooltip":{"show":true,"textStyle":{"color":"fff"}},"title":{"top":"75%","show":true,"x":"center","text":"","textStyle":{"color":"rgba(0,0,0,0.75)","fontSize":16,"fontWeight":"normal"}}}', '', '2019-11-07 11:27:24', NULL, 4, 25, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (44, 'otherindex', '指标数据', '指标数据', 'otherindex', '{"metadata":[{"displayname":"meta信息","lastupdate":"2021-10-15","fields":[{"fieldname":"地区编号","fieldkey":"districtid"},{"fieldname":"地区名","fieldkey":"districtname"},{"fieldname":"项目阶段","fieldkey":"scjdid"},{"fieldname":"更新时间","fieldkey":"updatetime"}],"dimension":["name","value"]}],"code":0,"data":[{"districtid":442000,"districtname":"中山市","children":[{"unit":"个","name":"项目数量","value":1111,"key":"xmsl"},{"unit":"亿","name":"投资额","value":2222,"key":"tze"}],"name":"标题一","updatetime":"2021-10-15"}],"count":1,"message":""}', 1, '{"value2":{"color":"#3c78dc","fontSize":"0.875rem"},"value1":{"color":"#3c78dc","fontSize":"1.25rem","marginLeft":"0.5rem"},"displayname":{"margin":"0.5rem 0 0 0.5rem","color":"color: rgba(0, 0, 0, 0.75)","fontSize":"1rem","lineHeight":"1.125rem"},"count":{"color":"#00b585","fontSize":"1.25rem"},"unit1":{"color":"rgba(0, 0, 0, 0.45)","fontSize":"0.875rem","marginLeft":"0.5rem"},"unit2":{"color":"rgba(0, 0, 0, 0.45)","fontSize":"0.875rem"},"remark":"当月","unit3":{"display":"none"},"time":{"padding":"0","backgroundColor":"#6499e8","margin":"0.375rem 0.5rem 0 0 ","color":"#fff","width":"3.25rem","lineHeight":"1.25rem","fontSize":"0.75rem","fontWeight":"400","height":"1.25rem","marginLeft":"auto"},"indext":{"border":"1px solid #f2f7f6","padding":"4px 2px 8px","borderRadius":"4px","height":"100%"}}', NULL, '2020-08-25 11:37:32', NULL, 6, 41159, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (86, 'barv-range', '条形排名图', '条形排名图', 'bar', '{"metadata":[{"displayname":"汇总信息","fields":[{"fieldname":"名称","fieldkey":"name"},{"fieldname":"数值","fieldkey":"value"}],"dimension":["name","value"]}],"code":0,"data":[{"name":"类目一","value":300},{"name":"类目二","value":250},{"name":"类目三","value":200},{"name":"类目四","value":150},{"name":"类目五","value":125},{"name":"类目六","value":100}],"count":6,"message":""}', 1, '{"yAxis":[{"axisLabel":{"color":"rgba(0,0,0,0.8)","rich":{"a":{"marginRight":"14px","backgroundColor":"#209F84","color":"#fff","borderRadius":50,"width":20,"fontSize":12,"align":"center","height":20},"b":{"color":"#000000","width":45,"fontSize":12,"align":"center","height":24}}},"inverse":true,"axisLine":{"show":false},"show":"false","splitLine":{"lineStyle":{"color":"rgba(0,0,0,0.1)"}},"axisTick":{"show":false},"axisPointer":{"show":true,"label":{"show":false}},"type":"category"},{"axisLabel":{"show":true,"textStyle":{"color":"rgba(0, 0, 0, 0.75)","fontSize":12},"inside":false},"data":[],"axisLine":{"show":false},"splitArea":{"show":false},"axisTick":{"show":false},"splitLine":{"show":false},"type":"category"}],"xAxis":{"axisLabel":{"color":"rgba(0,0,0,0.8)","show":false},"axisLine":{"show":false},"name":"公顷","axisTick":{"show":false},"splitLine":{"lineStyle":{"color":"rgba(0,0,0,0.06)"},"show":false},"nameGap":5,"nameLocation":"end","type":"value","nameTextStyle":{"color":"rgba(0,0,0,0.5)","fontSize":12}},"barMaxWidth":"12px","legend":{"data":["湿地"]},"grid":{"x":15,"y":8,"y2":8,"x2":10,"containLabel":true},"series":[{"showBackground":true,"backgroundStyle":{"color":"#EBEEF5"},"name":"面积","itemStyle":{"normal":{"color":{"x":0,"y":0,"y2":1,"x2":0,"global":false,"colorStops":[{"offset":0,"color":"#69D2B9"},{"offset":1,"color":"#69D2B9"}],"type":"linear"}}},"label":{"normal":{"color":"rgba(0, 0, 0, 0.75)","show":false,"fontSize":12,"position":"inside"}},"type":"bar"}],"limit":5,"tooltip":{"axisPointer":{"type":"shadow"},"trigger":"axis"},"rich":3,"dataset":{"source":[{"name":"第五分局","value":6635.29},{"name":"第四分局","value":5642.07},{"name":"第三分局","value":5543.34},{"name":"第二分局","value":4013.14},{"name":"第一分局","value":3803.53},{"name":"火炬分局","value":858.54}],"dimensions":["name","value"]}}', '', '2022-01-12 11:27:24', NULL, 4, 41236, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (87, 'traversal', '列表', '遍历组件', 'othertraversal', '{"metadata":[{"displayname":"汇总信息","fields":[{"fieldname":"名称","fieldkey":"name"},{"fieldname":"数值","fieldkey":"value"}],"dimension":[]}],"code":0,"data":[{"number":1,"image":"","unit":"分","trend":1,"grade":92,"icon":"icon-arrowright","title":"昆明市"},{"number":2,"image":"","unit":"分","trend":1,"grade":91,"icon":"icon-arrowright","title":"曲靖市"},{"number":3,"image":"","unit":"分","trend":0,"grade":87,"icon":"icon-arrowright","title":"玉溪市"},{"number":4,"image":"","unit":"分","trend":-1,"grade":78,"icon":"icon-arrowright","title":"临沧市"}]}', 1, '{"ulist":{"alignItems":"center","display":"flex","position":"relative","borderBottom":"1px solid #ccc","height":"88px"},"special":{"0":{"bgImage":"https://open.chinadci.com/onemap/api/filemeta/mix/18f6d278b87c19a60ebca7c80281c23b?isdownfile=0"},"2":{"bgImage":"/api/filemeta/mix/18f6d278b87c19a60ebca7c80281c23b?isdownfile=0"},"3":{"color":"#fff","background":"-webkit-linear-gradient(-45deg, #caff55 20%, #45cd00 60%)"}},"number":{"backgroundColor":"#d3e3ff","borderRadius":"20px","color":"#4583f0","textAlign":"center","left":"20px","width":"40px","lineHeight":"40px","backgroundSize":"cover","position":"absolute","height":"40px"},"image":{"left":"280px","width":"40px","fontSize":"26px","backgroundSize":"cover","position":"absolute","bgImage":"/api/filemeta/mix/18f6d278b87c19a60ebca7c80281c23b?isdownfile=0","height":"40px"},"unit":{"top":"35px","position":"absolute","right":"83px"},"trend":{"backgroundColor":"#f9bc73","width":"20px","position":"absolute","right":"47px","height":"8px"},"icon-arrow-down":{"-webkitTextFillColor":"transparent","background":"-webkit-linear-gradient(-90deg, #ccc 10%, #f00 80%)","position":"absolute","right":"53px","-webkitBackgroundClip":"text"},"grade":{"fontSize":"24px","position":"absolute","right":"103px","fontWeight":600},"icon-arrow-up":{"-webkitTextFillColor":"transparent","background":"-webkit-linear-gradient(-90deg, #45cd00 10%, #ccc 80%)","position":"absolute","right":"53px","-webkitBackgroundClip":"text"},"actionUrl":"/views/portrayal/948892764058918912?title=各市营业环境红黑榜","icon-arrowright":{"position":"absolute","right":"13px"},"title":{"left":"80px","fontSize":"26px","position":"absolute"}}', NULL, '2021-12-15 10:37:10', NULL, 42, 42246, NULL, NULL, NULL, 'other', NULL, 1);
INSERT INTO graphtemplate VALUES (56, 'subtitle', '副标题', '副标题', 'subtitle', '{"metadata":[{"lastupdate":"2022-01-16","fields":[{"fieldname":"年份","fieldkey":"year"}],"dimension":["name","value"]}],"code":0,"data":[{"children":[{"unit":"亩","name":"任务面积","value":10800,"key":"taskarea"},{"unit":"个","name":"任务个数","value":135,"key":"taskarea"}],"name":2021}],"count":2,"message":""}', 1, '{"container":{"padding":"0rem 0.75rem","margin":"0rem 0rem 1rem 0rem","borderRadius":"0.25rem","background":"#F7F7F7","lineHeight":"3.25rem","height":"3.25rem"},"num1":{"marginRight":"0.3125rem","color":"#28B1BF","fontSize":"1.125rem","fontWeight":"700"},"unit1":{"marginRight":"0.625rem","color":"rgba(0, 0, 0, 0.45)","fontSize":"0.75rem","fontWeight":"400"},"unit2":{"marginRight":"0rem","color":"rgba(0, 0, 0, 0.45)","fontSize":"0.75rem","fontWeight":"400"},"title":{"color":"rgba(0, 0, 0, 0.75)","fontSize":"0.875rem"},"num2":{"marginRight":"0.3125rem","color":"#28B1BF","fontSize":"1.125rem","fontWeight":"bold"}}', NULL, '2020-08-25 14:02:06', NULL, 6, 39524, NULL, NULL, NULL, 'text', NULL, 2);
INSERT INTO graphtemplate VALUES (88, 'dynamicAtlas', '图集', '动态图集', 'dynamicAtlas', '{"metadata":[{"displayname":"图片集","fields":[{"fieldname":"链接","fieldkey":"url"},{"fieldname":"标题","fieldkey":"title"}],"dimension":["url","title"]}],"code":0,"data":[{"title":"轮播图片一","url":"/api/filemeta/46602"},{"title":"轮播图片二","url":"/api/filemeta/46600"},{"title":"轮播图片三","url":"/api/filemeta/46599"},{"title":"轮播图片四","url":"/api/filemeta/46601"}]}', 1, '{"indicatorStyle":{"position":"absolute","bottom":"5px","right":"15px"},"indicatorType":"fraction","loop":true,"interval":3000,"carousel":{"height":"300px"},"autoplay":true}', NULL, '2021-12-15 10:37:10', NULL, 42, 46669, NULL, NULL, NULL, 'other', NULL, 1);
INSERT INTO graphtemplate VALUES (62, 'otherbars', '占比条形图', '占比条形图', 'otherbars', '{"code":0,"data":[{"unit":"%","name":"符合规划","value":"65"},{"unit":"%","name":"不符合规划","value":"35"}],"count":2,"message":"成功"}', 1, '{"bar":{"height":"30px"},"bargroup":{"display":"flex","height":"20px"},"barright":{"marginRight":"8px","marginLeft":"8px","whiteSpace":"nowrap","color":"#00b585","borderRadius":"0px 5px 5px 0px","fontSize":"12px","fontWeight":"bold"},"name":{"marginRight":"8px","color":"rgba(0, 0, 0, 0.75)","textAlign":"center","fontSize":"14px","fontWeight":"400"},"barleft":{"marginLeft":"8px","marginRight":"8px","whiteSpace":"nowrap","color":"#3c78dc","borderRadius":"3px 0px 0px 3px","fontSize":"12px","fontWeight":"bold"},"circle":{"marginRight":"8px","overflow":"hidden","borderRadius":"5px","width":"10px","height":"10px"},"colors":["#3c78dc","#00b585"],"leg":{"alignItems":"center","display":"flex","justifyContent":"center","height":"40px"}}', NULL, '2021-01-20 15:21:09', NULL, 6, 41161, NULL, NULL, NULL, 'index', NULL, 2);
INSERT INTO graphtemplate VALUES (75, 'tablefilter', '表格筛选', '表格筛选', 'tablefilter', '{"metadata":[{"displayname":"表格清单","fields":[{"fieldname":"时间","fieldkey":"displayName"},{"fieldname":"收件","unit":"件","fieldkey":"receive"},{"fieldname":"受理","unit":"件","fieldkey":"deal"},{"fieldname":"办结","unit":"件","fieldkey":"completed"},{"fieldname":"超期","unit":"件","fieldkey":"overdue"}]}],"code":0,"data":[{"receive":6,"totalDealtime":0,"deal":0,"overdue":0,"displayName":"当天","completed":0},{"receive":2394,"totalDealtime":90084.36,"deal":1668,"overdue":35,"displayName":"当月累计","completed":474},{"receive":9307,"totalDealtime":261659.43,"deal":6815,"overdue":185,"displayName":"当年累计","completed":3911},{"receive":31891,"totalDealtime":1747988.36,"deal":24602,"overdue":806,"displayName":"上线以来","completed":20389}],"count":4,"message":""}', 1, '{"border":false,"body1":{"backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","textAlign":"left","fontSize":"13px","fontWeight":400,"height":"40px"},"body2":{"backgroundColor":"#fafafa","color":"rgba(0,0,0,0.75)","textAlign":"left","fontSize":"13px","fontWeight":400,"height":"40px"},"header":{"margin":16,"backgroundColor":"#E4F3F0","color":"rgba(0,0,0,0.6)","textAlign":"left","fontWeight":"bold","height":"40px"},"cellstyle":{"1":{"filter":true,"sorter":true,"width":110,"align":"left"},"2":{"sorter":true,"width":110,"align":"left"},"textAlign":"left","width":80},"tablestyle":{"backgroundColor":"#fff","width":"100%","height":""}}', NULL, '2020-08-25 13:54:50', NULL, 11, 1423, NULL, NULL, NULL, 'table', NULL, 1);
INSERT INTO graphtemplate VALUES (89, 'dynamicAtlas', '图集测试', '图集测试', 'dynamicAtlas', '{"metadata":[{"displayname":"图片集","fields":[{"fieldname":"链接","fieldkey":"url"},{"fieldname":"标题","fieldkey":"title"}],"dimension":["url","title"]}],"code":0,"data":[{"title":"轮播图片一","url":"/api/filemeta/46602"},{"title":"轮播图片二","url":"/api/filemeta/46600"},{"title":"轮播图片三","url":"/api/filemeta/46599"},{"title":"轮播图片四","url":"/api/filemeta/46601"}]}', 1, '{"indicatorStyle":{"position":"absolute","bottom":"5px","right":"15px"},"indicatorType":"fraction","loop":true,"interval":3000,"carousel":{"height":"300px"},"autoplay":true}', NULL, NULL, NULL, 42, 46917, NULL, NULL, NULL, 'other', NULL, 1);
INSERT INTO graphtemplate VALUES (34, 'semicircle', '半圆双指标', '指标', 'semicircle', '{"code":0,"message":"","count":2,"metadata":[{"fields":[{"fieldkey":"taskarea","fieldname":"任务面积"},{"fieldkey":"name","fieldname":"地点"}],"dimension":["name","value"]}],"data":[{"name":"医院","value":10800},{"name":"学校","value":13500}]}', 1, '{"grid":{"bottom":"28px","containLabel":true},"series":[{"hoverAnimation":false,"legendHoverLink":false,"startAngle":180,"data":[],"clockwise":false,"center":["25%","78%"],"name":"1","label":{"show":false},"labelLine":{"show":false},"type":"pie","radius":["90%","130%"]},{"hoverAnimation":false,"legendHoverLink":false,"startAngle":180,"data":[],"clockwise":false,"center":["75%","78%"],"name":"2","label":{"normal":{"show":false},"emphasis":{"show":false}},"labelLine":{"normal":{"show":false}},"type":"pie","radius":["90%","130%"]}],"tooltip":{"show":false},"title":[{"subtext":"","textAlign":"center","x":"24%","y":"48%","text":"","textStyle":{"color":"#2857b5","fontSize":20},"subtextStyle":{"color":"rgba(0,0,0,0.7)","fontSize":14}},{"subtext":"","textAlign":"center","x":"75%","y":"48%","text":"","textStyle":{"color":"#008f6d","fontSize":20},"subtextStyle":{"color":"rgba(0,0,0,0.7)","fontSize":14}}]}', NULL, '2020-04-13 10:33:50', NULL, 4, 243, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (13, 'indexper', '百分比指标', NULL, 'indexper', '{"value":50,"name":"完成率"}', 1, '{"series":[{"hoverAnimation":true,"legendHoverLink":false,"startAngle":225,"color":[{"x":1,"y":0,"y2":0,"x2":0,"colorStops":[{"offset":0,"color":"#6499e8"},{"offset":1,"color":"#90bcf5"}],"type":"linear"},"transparent"],"data":[{"label":{"normal":{"formatter":"50%","show":true,"position":"center","textStyle":{"color":"#3c78dc","fontSize":"22","fontWeight":"bold"}}},"value":""},{"value":""}],"z":10,"labelLine":{"normal":{"show":false}},"type":"pie","radius":["50%","80%"]},{"silent":true,"startAngle":225,"data":[{"itemStyle":{"color":"#bfddff"},"value":75},{"itemStyle":{"color":"transparent"},"value":25}],"name":"","z":5,"labelLine":{"normal":{"show":false}},"type":"pie","radius":["50%","80%"]},{"cursor":"default","hoverAnimation":false,"legendHoverLink":false,"data":[{"value":1}],"name":"","itemStyle":{"color":{"x":0,"y":0,"y2":1,"x2":0,"colorStops":[{"offset":0,"color":"#f0f8ff"},{"offset":1,"color":"#fff"}],"type":"linear"}},"labelLine":{"normal":{"show":false}},"type":"pie","radius":["0","98%"]},{"cursor":"default","hoverAnimation":false,"legendHoverLink":false,"data":[{"itemStyle":{"normal":{"borderColor":{"x":0,"y":0,"y2":1,"x2":0,"colorStops":[{"offset":0,"color":"#bfddff"},{"offset":1,"color":"#e8f4ff"}],"type":"linear"},"borderWidth":2}},"value":1}],"name":"","z":5,"labelLine":{"normal":{"show":false}},"type":"pie","radius":["42%","42%"]}],"title":{"textAlign":"center","x":"50%","y":"70%","text":"","textStyle":{"color":"rgba(0,0,0,0.75)","fontSize":16,"fontWeight":"normal"}}}', NULL, NULL, NULL, 4, 38815, NULL, NULL, NULL, 'graph', NULL, 1);
INSERT INTO graphtemplate VALUES (74, 'tablelist', '表格清单', '表格清单', 'tablelist', '{"metadata":[{"displayname":"表格清单","fields":[{"fieldname":"时间","fieldkey":"displayName"},{"fieldname":"收件","unit":"件","fieldkey":"receive"},{"fieldname":"受理","unit":"件","fieldkey":"deal"},{"fieldname":"办结","unit":"件","fieldkey":"completed"},{"fieldname":"超期","unit":"件","fieldkey":"overdue"}]}],"code":0,"data":[{"receive":6,"totalDealtime":0,"deal":0,"overdue":0,"displayName":"当天","completed":0},{"receive":2394,"totalDealtime":90084.36,"deal":1668,"overdue":35,"displayName":"当月累计","completed":474},{"receive":9307,"totalDealtime":261659.43,"deal":6815,"overdue":185,"displayName":"当年累计","completed":3911},{"receive":31891,"totalDealtime":1747988.36,"deal":24602,"overdue":806,"displayName":"上线以来","completed":20389}],"count":4,"message":""}', 1, '{"table2":{"border":false,"body1":{"backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","textAlign":"left","fontSize":"13px","fontWeight":400,"height":"40px"},"body2":{"backgroundColor":"#fafafa","color":"rgba(0,0,0,0.75)","textAlign":"left","fontSize":"13px","fontWeight":400,"height":"40px"},"header":{"margin":16,"backgroundColor":"#E4F3F0","color":"rgba(0,0,0,0.6)","textAlign":"left","fontWeight":"bold","height":"40px"},"cellstyle":{"1":{"filter":true,"sorter":true,"width":135,"align":"left"},"4":{"sorter":true},"textAlign":"left","width":80},"tablestyle":{"backgroundColor":"#fff","width":"100%","height":""}},"table1":{"border":false,"footer":true,"body1":{"backgroundColor":"#fff","color":"rgba(0,0,0,0.75)","textAlign":"left","fontSize":"13px","fontWeight":400,"height":"40px"},"footerTitle":"查看全部","footerStyle":{"height":"40px"},"stripe":false,"body2":{"backgroundColor":"#fafafa","color":"rgba(0,0,0,0.75)","textAlign":"left","fontSize":"13px","fontWeight":400,"height":"40px"},"header":{"margin":16,"backgroundColor":"#E4F3F0","color":"rgba(0,0,0,0.6)","textAlign":"left","fontWeight":"bold","height":"40px"},"cellstyle":{"1":{"width":120,"align":"left"},"textAlign":"right","width":80},"tablestyle":{"backgroundColor":"#fff","width":"100%","height":""}}}', NULL, '2020-08-25 13:54:50', NULL, 11, 1423, NULL, NULL, NULL, 'table', NULL, 1);
COMMIT;
--创建graphtemplate_detail表
CREATE TABLE IF NOT EXISTS graphtemplate_detail (
  id int8 NOT NULL PRIMARY KEY,
  graphtemplate_id int8,
  type int4,
  content text,
  displayname varchar(255),
  thumbnail int8
);
-- ----------------------------
-- Records of graphtemplate_detail
-- ----------------------------
BEGIN;
INSERT INTO graphtemplate_detail VALUES (1, 3, 1, '{"dataset":[{"name":"类目一","value":30},{"name":"类目二","value":35},{"name":"类目三","value":40},{"name":"类目四","value":45},{"name":"类目五","value":50}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (2, 31, 1, '{"dataset":[{"value":5,"displayname":"90岁及以上","index":10},{"value":10,"displayname":"80-89岁","index":9},{"value":30,"displayname":"70-79岁","index":8},{"value":40,"displayname":"60-69岁","index":7},{"value":70,"displayname":"50-59岁","index":6},{"value":80,"displayname":"40-49岁","index":5},{"value":70,"displayname":"30-39岁","index":4},{"value":100,"displayname":"20-29岁","index":3},{"value":90,"displayname":"10-19岁","index":2},{"value":80,"displayname":"0-9岁","index":1}],"dimensions":["displayname","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (4, 32, 1, '{"dataset":[50,40]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (5, 20, 1, '{"dataset":[{"name":"类目五","value1":260,"value2":50},{"name":"类目四","value1":270,"value2":100},{"name":"类目三","value1":280,"value2":150},{"name":"类目二","value1":290,"value2":200},{"name":"类目一","value1":300,"value2":250}],"dimension":["name","value1","value2"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (6, 21, 1, '{"name":"类目分值","value":75}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (7, 13, 1, '{"name":"类目占比","value":80}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (8, 33, 1, '{"name":"类目占比","value":80}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (9, 27, 1, '{"dataset":[{"name":"类目一","value":10},{"name":"类目二","value":20},{"name":"类目三","value":30},{"name":"类目四","value":40},{"name":"类目五","value":50},{"name":"类目六","value":60}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (10, 2, 1, '{"dataset":[{"name":"类目一","value":40},{"name":"类目二","value":20},{"name":"类目三","value":40},{"name":"类目四","value":35},{"name":"类目五","value":50}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (11, 25, 1, '{"dataset":[{"name":"类目一","value":40},{"name":"类目二","value":20},{"name":"类目三","value":40},{"name":"类目四","value":35},{"name":"类目五","value":50}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (13, 1, 1, '{"dataset":[{"name":"类目一","value":50},{"name":"类目二","value":45},{"name":"类目三","value":40},{"name":"类目四","value":35},{"name":"类目五","value":30}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (14, 30, 1, '{"dataset":[{"name":"类目一","value":100},{"name":"类目二","value":50},{"name":"类目三","value":30},{"name":"类目四","value":15},{"name":"类目五","value":10}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (15, 17, 1, '{"dataset":[{"value2":30,"value1":45,"name":"类目一"},{"value2":25,"value1":35,"name":"类目二"},{"value2":50,"value1":35,"name":"类目三"},{"value2":25,"value1":35,"name":"类目四"},{"value2":30,"value1":40,"name":"类目五"},{"value2":50,"value1":30,"name":"类目六"}],"dimension":["name","value1","value2"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (16, 26, 1, '{"dataset":[{"xzmc":"类目一","zyl":100},{"xzmc":"类目二","zyl":90},{"xzmc":"类目三","zyl":80},{"xzmc":"类目四","zyl":70},{"xzmc":"类目五","zyl":60},{"xzmc":"类目六","zyl":50},{"xzmc":"类目七","zyl":40},{"xzmc":"类目八","zyl":30}],"dimension":["xzmc","zyl"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (17, 34, 1, '{"dataset":[{"displayname":"类目一（个）","value":100},{"displayname":"类目二（个）","value":120}],"dimension":["displayname","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (18, 18, 1, '{"dataset":[{"name":"类目一","value":100},{"name":"类目二","value":50},{"name":"类目三","value":30},{"name":"类目四","value":15},{"name":"类目五","value":10}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (20, 20, 3, '958', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (21, 3, 3, '959', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (22, 2, 3, '960', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (23, 25, 3, '960', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (12, 22, 1, '{"dataset":[{"name":"类目一","value":50},{"name":"类目二","value":45},{"name":"类目三","value":40},{"name":"类目四","value":35},{"name":"类目五","value":30}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (24, 1, 3, '962', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (32, 22, 3, '962', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (33, 30, 3, '963', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (28, 21, 3, '964', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (45, 18, 4, '1145', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (34, 33, 3, '964', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (27, 18, 3, '963', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (29, 31, 3, '966', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (25, 13, 3, '967', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (31, 27, 3, '968', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (26, 17, 3, '969', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (36, 32, 3, '970', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (30, 26, 3, '972', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (35, 34, 3, '973', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (40, 2, 4, '1138', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (50, 22, 4, '1136', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (3, 24, 1, '{"dataset":[{"name":"类目五","value":30},{"name":"类目四","value":35},{"name":"类目三","value":40},{"name":"类目二","value":45},{"name":"类目一","value":50}],"dimension":["name","value"]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (19, 24, 3, '1149', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (54, 32, 4, '1140', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (42, 1, 4, '1136', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (43, 13, 4, '1142', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (44, 17, 4, '1135', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (46, 21, 4, '1148', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (47, 31, 4, '1143', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (48, 26, 4, '1146', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (49, 27, 4, '1141', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (51, 30, 4, '1145', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (53, 34, 4, '1147', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (52, 33, 4, '1148', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (41, 25, 4, '1138', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (56, 14, 3, '1356', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (57, 14, 4, '1357', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (55, 14, 1, '{"code":0,"message":"","count":0,"metadata":[{"title":"表头一","dataIndex":"bt1","key":"bt1","width":100},{"title":"表头二","dataIndex":"bt2","key":"bt2","width":100},{"title":"表头三","dataIndex":"bt3","key":"bt3","width":100},{"title":"表头四","dataIndex":"bt4","key":"bt4","width":100},{"title":"表头五","dataIndex":"bt5","key":"bt5","width":100}],"data":[{"bt1":"区域一","bt2":10,"bt3":20,"bt4":30,"bt5":"状态一"},{"bt1":"区域二","bt2":30,"bt3":20,"bt4":10,"bt5":"状态二"}]}', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (37, 24, 4, '1137', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (38, 20, 4, '1144', NULL, NULL);
INSERT INTO graphtemplate_detail VALUES (39, 3, 4, '1139', NULL, NULL);
COMMIT;
```



### 4.8 专题目录接口

专题目录接口依赖**文件资源管理接口**、**地图数据资源管理接口**、**专题配置接口**、**画像接口**、**图表接口**。

专题目录接口需要以下数据表：***topiccatalog***、***topiccatalog_detail***表。

```sql
--创建topiccatalog表的ID序列
CREATE SEQUENCE IF NOT EXISTS topiccatalog_id_seq;
--创建topiccatalog表
CREATE TABLE IF NOT EXISTS topiccatalog (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('topiccatalog_id_seq'::regclass),
  pid int8,
  name varchar(255),
  description varchar(255),
  ispublish bool,
  type int2,
  content varchar(255),
  platsupport int8,
  nodetype int2,
  weight int4,
  icon varchar(255),
  icontype int2,
  thumbnail int8,
  displayname varchar(255),
  recommend int2,
  creator int8,
  createtime timestamp(6),
  updator int8,
  updatetime timestamp(6),
  pageviews int8,
  sindex int4
);
--创建topiccatalog_detail表的ID序列
CREATE SEQUENCE IF NOT EXISTS topiccatalog_detail_id_seq;
--创建topiccatalog_detail表
CREATE TABLE IF NOT EXISTS topiccatalog_detail (
	id int8 DEFAULT nextval('topiccatalog_detail_id_seq'::regclass),
	claimtype int2,
	content varchar(255),
	type int2,
	topiccatalogid int4
);
```



### 4.9 模型管理接口

模型管理接口需要以下数据表：***operatorcatalog***、***modelcatalog***、***modeldetail***、***operatorparams***、***modelio***、***modelanalysis***、***modelresult***表，这些数据表的创建SQL如下：

```sql
--创建operatorcatalog表的ID序列
CREATE SEQUENCE IF NOT EXISTS operatorcatalog_id_seq;
--创建operatorcatalog表
CREATE TABLE IF NOT EXISTS operatorcatalog (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('operatorcatalog_id_seq'::regclass),
  pid int8,
  displayname varchar,
  description varchar,
  nodetype int4,
  icon int8,
  identifier varchar,
  sindex int4,
  createtime timestamp(6),
  creator int8,
  updatetime timestamp(6),
  updator int8,
  isauthorize bool DEFAULT false
);
--创建modelcatalog表的ID序列
CREATE SEQUENCE IF NOT EXISTS modelcatalog_id_seq;
--创建modelcatalog表
CREATE TABLE IF NOT EXISTS modelcatalog (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('modelcatalog_id_seq'::regclass),
  pid int8,
  displayname varchar,
  description varchar,
  nodetype int4,
  modeldata varchar,
  thumbnail int8,
  sindex int4,
  createtime timestamp(6),
  creator int8,
  updatetime timestamp(6),
  updator int8,
  copyby int8,
  authmode int4 DEFAULT 0
);
--创建modeldetail表的ID序列
CREATE SEQUENCE IF NOT EXISTS modeldetail_id_seq;
--创建modeldetail表
CREATE TABLE IF NOT EXISTS modeldetail (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('modeldetail_id_seq'::regclass),
  pid int8,
  displayname varchar,
  referenceid int8,
  description varchar,
  sindex int4,
  originid varchar unique,
  createtime timestamp(6),
  creator int8,
  updatetime timestamp(6),
  updator int8,
  modelid int8
);
--创建operatorparams表的ID序列
CREATE SEQUENCE IF NOT EXISTS operatorparams_id_seq;
--创建operatorparams表
CREATE TABLE IF NOT EXISTS operatorparams (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('operatorparams_id_seq'::regclass),
  operatorid int8,
  name varchar,
  displayname varchar,
  description varchar,
  defaultvalue varchar,
  component varchar,
  style varchar,
  sindex int4,
  enumurl varchar
);
--设置operatorid字段外键关联
ALTER TABLE operatorparams ADD CONSTRAINT fk_operator_reference_operator FOREIGN KEY (operatorid) REFERENCES operatorcatalog (id) ON DELETE RESTRICT ON UPDATE RESTRICT;
--创建modelio表的ID序列
CREATE SEQUENCE IF NOT EXISTS modelio_id_seq;
--创建modelio表
CREATE TABLE IF NOT EXISTS modelio (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('modelio_id_seq'::regclass),
  modelid int8,
  detailid int8,
  paramtype int4,
  paramvalue varchar,
  paramname varchar,
  originid varchar,
  createtime timestamp(6),
  creator int8,
  updatetime timestamp(6),
  updator int8,
  globalname varchar,
  originvalue varchar
);
--设置唯一约束
ALTER TABLE modelio ADD CONSTRAINT modelio_unique_cons UNIQUE (detailid, originid);
--设置modelid字段的外键关联
ALTER TABLE modelio ADD CONSTRAINT fk_modelino_reference_modelcat FOREIGN KEY (modelid) REFERENCES modelcatalog (id) ON DELETE RESTRICT ON UPDATE RESTRICT;
--设置detailid字段的外键关联
ALTER TABLE modelio ADD CONSTRAINT fk_modelino_reference_modeldet FOREIGN KEY (detailid) REFERENCES modeldetail (id) ON DELETE RESTRICT ON UPDATE RESTRICT;
--创建modelanalysis表的ID序列
CREATE SEQUENCE IF NOT EXISTS modelanalysis_id_seq;
--创建modelanalysis表
CREATE TABLE IF NOT EXISTS modelanalysis (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('modelanalysis_id_seq'::regclass),
  modelid int8,
  status int4,
  message varchar,
  params varchar,
  operators int8[],
  operatornow int8,
  executor int8,
  starttime timestamp(6),
  updatetime timestamp(6),
  displayname varchar,
  description varchar,
  jobid int4,
  cronspec varchar(255)
);
--设置modelid字段的外键关联
ALTER TABLE modelanalysis ADD CONSTRAINT fk_modelana_reference_modelcat FOREIGN KEY (modelid) REFERENCES modelcatalog (id) ON DELETE RESTRICT ON UPDATE RESTRICT;
--创建modelresult表的ID序列
CREATE SEQUENCE IF NOT EXISTS modelresult_id_seq;
--创建modelresult表
CREATE TABLE IF NOT EXISTS modelresult (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('modelresult_id_seq'::regclass),
  analysisid int8,
  paramname varchar(255),
  globalname varchar(255),
  result varchar(255),
  createtime timestamp(6),
  updatetime timestamp(6),  
  tableid int8,
  mapserverid int8,
  resultname varchar(255),
  temporary varchar(255)
);
--设置唯一约束
ALTER TABLE modelresult ADD CONSTRAINT modelresult_unique_cons UNIQUE (analysisid, globalname);
--设置analysisid字段的外键关联
ALTER TABLE modelresult ADD CONSTRAINT fk_modelres_reference_modelana FOREIGN KEY (analysisid) REFERENCES modelanalysis (id) ON DELETE RESTRICT ON UPDATE RESTRICT;
```



### 4.10 数据表管理接口

数据表管理接口需要以下数据表：***dbtable***、***dbtable_field***、***d_unit_type***表，这些数据表的创建SQL如下：

```sql
--创建modelresult表的ID序列
CREATE SEQUENCE IF NOT EXISTS dbtable_id_seq;
--创建dbtable表
CREATE TABLE IF NOT EXISTS dbtable (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('dbtable_id_seq'::regclass),
  tablename varchar(255),
  displayname varchar(255),
  description varchar(255),
  schema varchar(255),
  viewortable int2,
  tabletype int8,
  hasshape bool,
  srid int8,
  type varchar(255),
  adcode int8,
  mainorganization varchar(255),
  status int2,
  weight int4,
  createtime timestamp(6),
  updatetime timestamp(6),
  deletetime timestamp(6) NOT NULL DEFAULT '9999-12-31 23:59:59'::timestamp without time zone,
  creator int8,
  updator int8,
  deletor int8,
  datasource int2,
  filemetaid int8
);
--设置唯一约束
ALTER TABLE dbtable ADD CONSTRAINT dbtable_unique_cons UNIQUE (tablename, schema, deletetime);
--创建dbtable_field表的ID序列
CREATE SEQUENCE IF NOT EXISTS dbtable_field_id_seq;
--创建dbtable_field表
CREATE TABLE IF NOT EXISTS dbtable_field (
	id int8 NOT NULL PRIMARY KEY DEFAULT nextval('dbtable_field_id_seq'::regclass),
	dbtableid int8,
	fieldname varchar(255),
	fielddisplayname varchar(255),
	fieldtype varchar(255),
	length int4,
	precision int4,
	fielddescription varchar(255),
	showfield bool,
	type varchar(255),
	weight int8,
	createtime timestamp(6),
	updatetime timestamp(6),
	deletetime timestamp(6) NOT NULL DEFAULT '9999-12-31 23:59:59'::timestamp without time zone,
	creator int8,
	updator int8,
	deletor int8,
	unit int4
);
--设置唯一约束
ALTER TABLE dbtable_field ADD CONSTRAINT dbtable_field_unique_cons UNIQUE (dbtableid, fieldname, deletetime);
--创建单位类型维度表
CREATE TABLE IF NOT EXISTS d_unit_type (
	id int4 NOT NULL PRIMARY KEY,
	unitname varchar,
	unittype varchar,
	englishname varchar,
	factor float8,
	primarytype bool DEFAULT false
);
-- ----------------------------
-- Records of d_unit_type
-- ----------------------------
BEGIN;
INSERT INTO d_unit_type VALUES (161011, '‰', '比例单位', 'ONE_OF_THOUSAND', 0.001, 'f');
INSERT INTO d_unit_type VALUES (161012, '%', '比例单位', 'ONE_OF_HUNDRED', 0.01, 'f');
INSERT INTO d_unit_type VALUES (121015, '公里', '长度单位', 'GONGLI', 1000, 'f');
INSERT INTO d_unit_type VALUES (111010, '分', '货币单位', 'FEN', 0.01, 'f');
INSERT INTO d_unit_type VALUES (111011, '角', '货币单位', 'JIAO', 0.1, 'f');
INSERT INTO d_unit_type VALUES (111012, '元', '货币单位', 'YUAN', 1, 't');
INSERT INTO d_unit_type VALUES (111412, '万元', '货币单位', 'TEN_THOUSAND_YUAN', 10000, 'f');
INSERT INTO d_unit_type VALUES (121415, '万公里', '长度单位', 'TEN_THOUSAND_GONGLI', 10000000, 'f');
INSERT INTO d_unit_type VALUES (131010, '平方毫米', '面积单位', 'SQUARE_MILLIMETER', 1e-06, 'f');
INSERT INTO d_unit_type VALUES (131011, '平方厘米', '面积单位', 'SQUARE_CENTIMETER', 0.0001, 'f');
INSERT INTO d_unit_type VALUES (121010, '毫米', '长度单位', 'MILLIMETER', 0.001, 'f');
INSERT INTO d_unit_type VALUES (121011, '厘米', '长度单位', 'CENTIMETER', 0.01, 'f');
INSERT INTO d_unit_type VALUES (121012, '分米', '长度单位', 'DECIMETER', 0.1, 'f');
INSERT INTO d_unit_type VALUES (121013, '米', '长度单位', 'METER', 1, 't');
INSERT INTO d_unit_type VALUES (121014, '千米', '长度单位', 'KILOMETER', 1000, 'f');
INSERT INTO d_unit_type VALUES (121016, '寸', '长度单位', 'CUN', 0.0333333, 'f');
INSERT INTO d_unit_type VALUES (121017, '尺', '长度单位', 'CHI', 0.3333333, 'f');
INSERT INTO d_unit_type VALUES (121018, '丈', '长度单位', 'ZHANG', 3.3333333, 'f');
INSERT INTO d_unit_type VALUES (121019, '里', '长度单位', 'LI', 500, 'f');
INSERT INTO d_unit_type VALUES (111512, '十万元', '货币单位', 'HUNDRED_THOUSAND_YUAN', 100000, 'f');
INSERT INTO d_unit_type VALUES (111612, '百万元', '货币单位', 'MILLION_YUAN', 1000000, 'f');
INSERT INTO d_unit_type VALUES (111712, '千万元', '货币单位', 'TEN_MILLION_YUAN', 10000000, 'f');
INSERT INTO d_unit_type VALUES (111812, '亿元', '货币单位', 'HUNDRED_MILLION_YUAN', 100000000, 'f');
INSERT INTO d_unit_type VALUES (161010, '‱', '比例单位', 'ONE_OF_TEN_THOUSAND', 0.0001, 'f');
INSERT INTO d_unit_type VALUES (121414, '万千米', '长度单位', 'TEN_THOUSAND_KILOMETER', 10000000, 'f');
INSERT INTO d_unit_type VALUES (161013, '比例', '比例单位', 'ONE_OF_ONE', 1, 't');
INSERT INTO d_unit_type VALUES (131413, '万平方米', '面积单位', 'TEN_THOUSAND_SQUARE_METER', 10000, 'f');
INSERT INTO d_unit_type VALUES (211010, '个', '个数单位', 'GE', 1, 't');
INSERT INTO d_unit_type VALUES (121413, '万米', '长度单位', 'TEN_THOUSAND_METER', 10000, 'f');
INSERT INTO d_unit_type VALUES (131414, '万公亩', '面积单位', 'TEN_THOUSAND_ARE', 1000000, 'f');
INSERT INTO d_unit_type VALUES (131415, '万公顷', '面积单位', 'TEN_THOUSAND_HECTARE', 100000000, 'f');
INSERT INTO d_unit_type VALUES (131416, '万平方千米', '面积单位', 'TEN_THOUSAND_SQUARE_KILOMETER', 10000000000, 'f');
INSERT INTO d_unit_type VALUES (131417, '万平方公里', '面积单位', 'TEN_THOUSAND_SQUARE_GONGLI', 10000000000, 'f');
INSERT INTO d_unit_type VALUES (131420, '万亩', '面积单位', 'TEN_THOUSAND_MU', 6666666.6666667, 'f');
INSERT INTO d_unit_type VALUES (131421, '万顷', '面积单位', 'TEN_THOUSAND_QING', 666666666.666667, 'f');
INSERT INTO d_unit_type VALUES (211410, '万个', '个数单位', 'TEN_THOUSAND_GE', 10000, 'f');
INSERT INTO d_unit_type VALUES (141413, '万立方米', '体积单位', 'TEN_THOUSAND_CUBIC_METER', 10000, 'f');
INSERT INTO d_unit_type VALUES (141414, '万立方千米', '体积单位', 'TEN_THOUSAND_CUBIC_KILOMETER', 10000000000000, 'f');
INSERT INTO d_unit_type VALUES (141416, '万升', '体积单位', 'TEN_THOUSAND_LITER', 10, 'f');
INSERT INTO d_unit_type VALUES (221010, '件', '件数单位', 'JIAN', 1, 't');
INSERT INTO d_unit_type VALUES (151414, '万吨', '重量单位', 'TEN_THOUSAND_TON', 10000000000, 'f');
INSERT INTO d_unit_type VALUES (151417, '万斤', '重量单位', 'TEN_THOUSAND_JIN', 5000000, 'f');
INSERT INTO d_unit_type VALUES (151418, '万公斤', '重量单位', 'TEN_THOUSAND_GONGJIN', 10000000, 'f');
INSERT INTO d_unit_type VALUES (151413, '万千克', '重量单位', 'TEN_THOUSAND_KILOGRAM', 10000000, 'f');
INSERT INTO d_unit_type VALUES (231010, '宗', '宗数单位', 'ZONG', 1, 't');
INSERT INTO d_unit_type VALUES (151419, '万担', '重量单位', 'TEN_THOUSAND_DAN', 500000000, 'f');
INSERT INTO d_unit_type VALUES (241010, '幢', '房屋数量单位', 'ZHUANG', 1, 't');
INSERT INTO d_unit_type VALUES (241410, '万幢', '房屋数量单位', 'TEN_THOUSAND_ZHUANG', 10000, 'f');
INSERT INTO d_unit_type VALUES (171010, '纳秒', '时间单位', 'NANOSECOND', 1e-09, 'f');
INSERT INTO d_unit_type VALUES (171011, '微秒', '时间单位', 'MICROSECOND', 1e-06, 'f');
INSERT INTO d_unit_type VALUES (171012, '毫秒', '时间单位', 'MILLISECOND', 0.001, 'f');
INSERT INTO d_unit_type VALUES (171013, '秒', '时间单位', 'SECOND', 1, 't');
INSERT INTO d_unit_type VALUES (171014, '分', '时间单位', 'MINUTE', 60, 'f');
INSERT INTO d_unit_type VALUES (171015, '时', '时间单位', 'HOUR', 3600, 'f');
INSERT INTO d_unit_type VALUES (171016, '天', '时间单位', 'DAY', 86400, 'f');
INSERT INTO d_unit_type VALUES (171017, '周', '时间单位', 'WEEK', 604800, 'f');
INSERT INTO d_unit_type VALUES (151013, '千克', '重量单位', 'KILOGRAM', 1000, 'f');
INSERT INTO d_unit_type VALUES (171018, '年', '时间单位', 'YEAR', 31536000, 'f');
INSERT INTO d_unit_type VALUES (151010, '毫克', '重量单位', 'MILLIGRAM', 0.001, 'f');
INSERT INTO d_unit_type VALUES (151011, '克拉', '重量单位', 'CARAT', 0.2, 'f');
INSERT INTO d_unit_type VALUES (151012, '克', '重量单位', 'GRAM', 1, 't');
INSERT INTO d_unit_type VALUES (151014, '吨', '重量单位', 'TON', 1000000, 'f');
INSERT INTO d_unit_type VALUES (151015, '钱', '重量单位', 'QIAN', 5, 'f');
INSERT INTO d_unit_type VALUES (151016, '两', '重量单位', 'LIANG', 50, 'f');
INSERT INTO d_unit_type VALUES (151017, '斤', '重量单位', 'JIN', 500, 'f');
INSERT INTO d_unit_type VALUES (151018, '公斤', '重量单位', 'GONGJIN', 1000, 'f');
INSERT INTO d_unit_type VALUES (151019, '担', '重量单位', 'DAN', 50000, 'f');
INSERT INTO d_unit_type VALUES (141010, '立方毫米', '体积单位', 'CUBIC_MILLIMETER', 1e-09, 'f');
INSERT INTO d_unit_type VALUES (141011, '立方厘米', '体积单位', 'CUBIC_CENTIMETER', 1e-06, 'f');
INSERT INTO d_unit_type VALUES (141012, '立方分米', '体积单位', 'CUBIC_DECIMETER', 0.001, 'f');
INSERT INTO d_unit_type VALUES (141013, '立方米', '体积单位', 'CUBIC_METER', 1, 't');
INSERT INTO d_unit_type VALUES (141014, '立方千米', '体积单位', 'CUBIC_KILOMETER', 1000000000, 'f');
INSERT INTO d_unit_type VALUES (141015, '毫升', '体积单位', 'MILLILITER', 1e-06, 'f');
INSERT INTO d_unit_type VALUES (141016, '升', '体积单位', 'LITER', 0.001, 'f');
INSERT INTO d_unit_type VALUES (131012, '平方分米', '面积单位', 'SQUARE_DECIMETER', 0.01, 'f');
INSERT INTO d_unit_type VALUES (131013, '平方米', '面积单位', 'SQUARE_METER', 1, 't');
INSERT INTO d_unit_type VALUES (131014, '公亩', '面积单位', 'ARE', 100, 'f');
INSERT INTO d_unit_type VALUES (131015, '公顷', '面积单位', 'HECTARE', 10000, 'f');
INSERT INTO d_unit_type VALUES (131016, '平方千米', '面积单位', 'SQUARE_KILOMETER', 1000000, 'f');
INSERT INTO d_unit_type VALUES (131017, '平方公里', '面积单位', 'SQUARE_GONGLI', 1000000, 'f');
INSERT INTO d_unit_type VALUES (131018, '平方寸', '面积单位', 'SQUARE_CUN', 0.0011111, 'f');
INSERT INTO d_unit_type VALUES (131019, '平方尺', '面积单位', 'SQUARE_CHI', 0.1111111, 'f');
INSERT INTO d_unit_type VALUES (131020, '亩', '面积单位', 'MU', 666.6666667, 'f');
INSERT INTO d_unit_type VALUES (131021, '顷', '面积单位', 'QING', 66666.6666667, 'f');
INSERT INTO d_unit_type VALUES (221410, '万件', '件数单位', 'TEN_THOUSAND_JIAN', 10000, 'f');
INSERT INTO d_unit_type VALUES (231410, '万宗', '宗数单位', 'TEN_THOUSAND_ZONG', 10000, 'f');
COMMIT;
--创建单位转换方法
CREATE OR REPLACE FUNCTION convertunit(numeric, integer, integer) RETURNS numeric
AS 'SELECT cast($1 * t1.factor / t2.factor AS numeric) AS newvalue
FROM d_unit_type t1, d_unit_type t2
WHERE t1.id=$2 AND t2.id=$3 AND t1.id/10000=t2.id/10000;'
LANGUAGE SQL
IMMUTABLE
RETURNS NULL ON NULL INPUT;
```



### 4.11 指标接口
指标接口依赖**专题目录接口**。

指标接口需要以下数据表：***indexmeta***、***indexcatalog***、***indexcatalogmeta***、***index_catalog_rel***、***indexevaluate_catalog***、***indexevaluatemethod***表。

```sql
--创建indexmeta表
CREATE TABLE IF NOT EXISTS indexmeta (
  id int8 NOT NULL PRIMARY KEY,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  indextable varchar(255),
  weight int8,
  type int2,
  spacecode varchar(255),
  spacetype varchar(255)
);
--创建indexcatalog表的ID序列
CREATE SEQUENCE IF NOT EXISTS indexcatalog_id_seq;
--创建indexcatalog表
CREATE TABLE IF NOT EXISTS indexcatalog (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('indexcatalog_id_seq'::regclass),
  pid int8,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  weight float8,
  indexmetaid int8,
  indexcatalogmetaid int4
);
--创建indexcatalogmeta表的ID序列
CREATE SEQUENCE IF NOT EXISTS indexcatalogmeta_id_seq;
--创建indexcatalogmeta表
CREATE TABLE IF NOT EXISTS indexcatalogmeta (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('indexcatalogmeta_id_seq'::regclass),
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  indexmetaid int4,
  weight float8
);
--创建index_catalog_rel表的ID序列
CREATE SEQUENCE IF NOT EXISTS index_catalog_rel_id_seq;
--创建index_catalog_rel表
CREATE TABLE IF NOT EXISTS index_catalog_rel (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('index_catalog_rel_id_seq'::regclass),
  indexid int8,
  indexcatalogid int8,
  weight int4,
  indextable varchar(255)
);
--创建indexevaluate_catalog表的ID序列
CREATE SEQUENCE IF NOT EXISTS indexevaluatecatalog_id_seq;
--创建indexevaluate_catalog表
CREATE TABLE IF NOT EXISTS indexevaluate_catalog (
  id int4 NOT NULL DEFAULT nextval('indexevaluatecatalog_id_seq'::regclass),
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  indexcatalogmetaid int4,
  year varchar(32),
  indexevaluatemethodid int4,
  spacecode varchar(255),
  createtime timestamp(0),
  creator int4,
  updatetime timestamp(0),
  updator int4
);
--创建indexevaluatemethod表
CREATE TABLE IF NOT EXISTS indexevaluatemethod (
  id int8 NOT NULL PRIMARY KEY,
  indexcatalogmetaid int4,
  name varchar(255),
  description varchar(255),
  weightmethod int4,
  normalizationmethod int4,
  type int4,
  displayname varchar(255),
  creator varchar(255),
  fileid int4,
  isdeleted bool
);
-- ----------------------------
-- Records of indexevaluatemethod
-- ----------------------------
BEGIN;
INSERT INTO indexevaluatemethod VALUES (4, 1, '极值归一化评价法', '基于极值进行指标归一化处理，归一化值乘以100得到指标分数。', NULL, NULL, 1, '极值归一化评价法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (5, 1, 'Z-score归一化评价法', '基于Z-score进行指标归一化处理，归一化值乘以100得到指标分数。', NULL, NULL, 1, 'Z-score归一化评价法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (6, 1, '极差值归一化评价法', '基于极差值进行指标归一化处理，归一化值乘以100得到指标分数。', NULL, NULL, 1, '极差值归一化评价法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (8, 1, '熵值评价法二', '基于熵值法确定各级指标的权重，并采用极值法进行指标值归一化处理，指标归一化值与指标权重加权求和得到指标分数。', 13, 15, 1, '熵值评价法二', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (19, 1, '均方差权重法', '基于均方差法原理确定权重的方法。', NULL, NULL, 2, '均方差权重法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (9, 1, '均方差评价法一', '基于均方差法确定各级指标的权重，并采用极值法进行指标值归一化处理，指标归一化值与指标权重加权求和得到指标分数。', 19, 15, 1, '均方差评价法一', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (11, 1, '主成分分析评价法一', '基于主成分分析法确定各级指标的权重，并采用极差值法进行指标值归一化处理，指标归一化值与指标权重加权求和得到指标分数。', 14, 16, 1, '主成分分析评价法一', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (17, 1, 'Z-score归一化', '基于Z-score法原理进行归一化处理。', NULL, NULL, 3, 'Z-score归一化', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (16, 1, '极差值归一化', '基于极差值法原理进行归一化处理。', NULL, NULL, 3, '极差值归一化', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (15, 1, '极值归一化', '基于极值法原理进行归一化处理。', NULL, NULL, 3, '极值归一化', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (14, 1, '主成分分析权重法', '基于主成分分析法原理确定权重的方法。', NULL, NULL, 2, '主成分分析权重法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (13, 1, '熵值权重法', '基于熵值法原理确定权重的方法。', NULL, NULL, 2, '熵值权重法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (12, 1, '层次分析权重法', '基于层次分析法原理确定权重的方法。', NULL, NULL, 2, '层次分析权重法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (10, 1, '专家打分法', '由少数专家直接根据经验并考虑反映某评价观点后直接对指标进行评价打分的方法', NULL, NULL, 1, '专家打分法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (7, 1, '客观评价法', '基于熵值法确定各级指标的权重，并采用极差值法进行指标值归一化处理，指标归一化值与指标权重加权求和得到指标分数。假设评价对象有m个，评价指标有n个，则第i个评价对象的第j个评价指标为 Xij 。计算步骤包括：1、数据标准化；2、计算第j项指标下第i个对象所占的比重 Zij；3、计算第j项指标的熵值；4、计算信息熵冗余度；5、计算各项指标的权重；6、计算各评价对象的综合得分。', 13, 16, 1, '客观评价法', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (1, 1, '层次分析法', '根据指标评价目标和指标体系按照各个指标因子间的相互关联影响以及隶属关系将其按不同的层次聚集组合，形成一个多层次的分析结构模型，从而根据底层指标相对于评价目标的相对重要权值和优劣进行指标总体评估。主要包括：（1）基于极差值法进行指标值归一化；（2）根据指标体系建立层次结构模型；（3）构造判断(成对比较)矩阵；（4）层次单排序及其一致性检验；（5）层次总排序及其一致性检验；（6）求指标的权重；（7）指标打分。', 12, 16, 1, '层次分析法', NULL, 5936, NULL);
INSERT INTO indexevaluatemethod VALUES (2, 1, '层次分析评价法二', '基于层次分析法确定各级指标的权重，并采用极值进行指标值归一化处理，指标归一化值与指标权重加权求和得到指标分数。', 12, 15, 1, '层次分析评价法二', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (3, 1, '层次分析评价法三', '基于层次分析法确定各级指标的权重，并采用Z-score法进行指标值归一化处理，指标归一化值与指标权重加权求和得到指标分数。', 12, 17, 1, '层次分析评价法三', NULL, NULL, NULL);
INSERT INTO indexevaluatemethod VALUES (18, 1, '专家评价法', '由少数专家直接根据经验并考虑反映某评价观点后定出指标分值的方法。专家评价法的主要步骤是：首先根据评价对象的具体情况选定评价指标，对每个指标均定出评价等级，每个等级的标准用分值表示；然后以此为基准，由专家对评价对象进行分析和评价，确定各个指标的分值，采用加法评分法、乘法评分法或加乘评分法求出个评价对象的总分值，从而得到评价结果。', NULL, NULL, 2, '专家评价法', NULL, NULL, NULL);
COMMIT;
```



### 4.12 查询统计分析接口

查询统计分析接口依赖**地图数据资源管理接口**、**图表接口**、**文件资源管理接口**、**指标接口**。

查询统计分析接口需要以下数据表：***layer_detail***表。

```sql
--创建layer_detail表的ID序列
CREATE SEQUENCE IF NOT EXISTS layer_detail_id_seq;
--创建layer_detail表
CREATE TABLE IF NOT EXISTS layer_detail (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('layer_detail_id_seq'::regclass),
  layerid int8,
  detailtype varchar(255),
  detailid int4,
  weight int4,
  creator int4,
  creattime timestamp(6),
  ispublic bool,
  likecount varchar(255)
);
```



### 4.13 用户消息接口

用户消息接口需要以下数据表：***usermessage***、***usermessage_status***表。

```sql
--创建用户消息表的ID序列
CREATE SEQUENCE IF NOT EXISTS usermessage_id_seq;
--创建用户消息表
CREATE TABLE IF NOT EXISTS usermessage (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('usermessage_id_seq'::regclass),
  pid int8,
  title varchar,
  description varchar,
  type int4,
  body varchar,
  related varchar,
  platform int8,
  sender int8,
  receiver int8[],
  sendtime timestamp(6)
);
--创建用户消息状态表的ID序列
CREATE SEQUENCE IF NOT EXISTS usermessage_status_id_seq;
--创建用户消息状态表
CREATE TABLE usermessage_status (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('usermessage_status_id_seq'::regclass),
  userid int8,
  messageid int8,
  status int4
);
--Foreign Keys structure for table usermessage_status
ALTER TABLE usermessage_status ADD CONSTRAINT fk_usermess_reference_usermess FOREIGN KEY (messageid) REFERENCES usermessage (id) ON DELETE RESTRICT ON UPDATE RESTRICT;
--设置唯一索引
ALTER TABLE usermessage_status ADD CONSTRAINT unique_usermessage_status_userid_messageid unique(userid, messageid);
```



### 4.14 数据检查与处理接口

数据检查与处理接口需要以下数据表：***dataupdate_items***、***dataupdate_jobs***、***dataupdate_results***表。

```sql
--创建dataupdate_items表
CREATE TABLE IF NOT EXISTS dataupdate_items (
  id int4 NOT NULL PRIMARY KEY,
  pid int4,
  displayname varchar(255),
  api varchar(255),
  isdefault bool,
  icon varchar(255),
  object varchar(255)
);
-- ----------------------------
-- Records of dataupdate_items
-- ----------------------------
BEGIN;
INSERT INTO dataupdate_items VALUES (1, NULL, '地图服务', NULL, NULL, NULL, NULL);
INSERT INTO dataupdate_items VALUES (2, NULL, '数据库', NULL, NULL, NULL, NULL);
INSERT INTO dataupdate_items VALUES (3, 1, '服务结构属性和空间参考变化', NULL, NULL, NULL, NULL);
INSERT INTO dataupdate_items VALUES (4, 1, '未注册成为服务的空间数据表', NULL, NULL, NULL, NULL);
INSERT INTO dataupdate_items VALUES (5, 2, '表结构和空间参考变化', NULL, NULL, NULL, NULL);
INSERT INTO dataupdate_items VALUES (6, 2, '新增表', NULL, NULL, NULL, NULL);
COMMIT;
--创建dataupdate_jobs表的ID序列
CREATE SEQUENCE IF NOT EXISTS dataupdate_jobs_id_seq;
--创建dataupdate_jobs表
CREATE TABLE IF NOT EXISTS dataupdate_jobs (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('dataupdate_jobs_id_seq'::regclass),
  dataupdate_itemid int8,
  starttime timestamp(6),
  stoptime timestamp(6),
  status int2,
  abnormalsize int4,
  creator varchar(255),
  datasize int4,
  testedsize int4,
  description varchar(255)
);
--创建dataupdate_results表的ID序列
CREATE SEQUENCE IF NOT EXISTS dataupdate_results_id_seq;
--创建dataupdate_results表
CREATE TABLE IF NOT EXISTS dataupdate_results (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('dataupdate_results_id_seq'::regclass),
  dataupdate_jobid int8,
  content varchar(255),
  createtime timestamp(6),
  handletime timestamp(6),
  status int2,
  handler int8,
  type varchar(255),
  description varchar(255)
);
```



### 4.15 文章管理接口

文章管理接口需要以下数据表：***articlemeta***表。

```sql
--创建articlemeta表的ID序列
CREATE SEQUENCE IF NOT EXISTS articlemeta_id_seq;
--创建articlemeta表
CREATE TABLE IF NOT EXISTS articlemeta (
  id int8 NOT NULL PRIMARY KEY DEFAULT nextval('articlemeta_id_seq'::regclass),
  displayname varchar(255),
  description varchar(255),
  thumbnail int8,
  type int8,
  content text,
  url varchar(255),
  creator int8,
  creattime timestamp(6),
  updator int8,
  updatetime timestamp(6),
  views int4
);
```



### 4.16 报告管理接口

报告管理接口需要以下数据表：***report***、***reporttemplate***、***reporttemplate_detail***表。

```sql
--创建report表的ID序列
CREATE SEQUENCE IF NOT EXISTS report_id_seq;
--创建report表
CREATE TABLE IF NOT EXISTS report (
  id int4 NOT NULL PRIMARY KEY DEFAULT nextval('report_id_seq'::regclass),
  fileid int4,
  name varchar(255),
  displayname varchar(255),
  contentpath varchar(255),
  template int8,
  createtime timestamp(0),
  updatetime timestamp(0),
  isdeleted bool,
  description varchar(255),
  filesize int8,
  views int8,
  downloads int8,
  creator int8,
  updator int8
);
--创建reporttemplate表的ID序列
CREATE SEQUENCE IF NOT EXISTS reporttemplate_id_seq;
--创建reporttemplate表
CREATE TABLE IF NOT EXISTS reporttemplate (
  id int4 NOT NULL PRIMARY KEY DEFAULT nextval('reporttemplate_id_seq'::regclass),
  fileid int4,
  name varchar(255),
  displayname varchar(255),
  description varchar(255),
  contentpath varchar(255),
  suffix varchar(255),
  copyby int4,
  createtime timestamp(6),
  creator int4,
  updatetime timestamp(6),
  updator int4,
  createtype int2,
  filesize int8,
  views int8,
  downloads int8,
  generates int8
);
--创建reporttemplate_detail表的ID序列
CREATE SEQUENCE IF NOT EXISTS reporttemplate_detail_id_seq;
--创建reporttemplate_detail表
CREATE TABLE IF NOT EXISTS reporttemplate_detail (
  id int4 NOT NULL PRIMARY KEY DEFAULT nextval('reporttemplate_detail_id_seq'::regclass),
  reporttemplateid int4,
  name varchar(255),
  description varchar(255),
  content varchar(255),
  prefix varchar(255),
  type int8,
  creator int4,
  createtime timestamp(6),
  updator int4,
  updatetime timestamp(6)
);
```



### 4.17 评论点赞接口

评论点赞接口需要usercomment、contentprefer_statistic表，依赖contenttype、user表。

```sql
--创建评论表的ID序列
CREATE SEQUENCE IF NOT EXISTS usercomment_id_seq;
--创建评论usercomment表
CREATE TABLE IF NOT EXISTS usercomment (
	id int8 NOT NULL PRIMARY KEY DEFAULT nextval('usercomment_id_seq'::regclass),
	pid int8,
	contenttype varchar(255),
	content varchar(255),
	comment varchar,
	ipaddress varchar(15),
	creator int8,
	createtime timestamptz(6),
	updatetime timestamptz(6),
	flag int2,
	weight int4
);
--创建内容偏好统计表的ID序列
CREATE SEQUENCE IF NOT EXISTS contentpreferstatistic_id_seq;
--创建内容偏好统计contentprefer_statistic表
CREATE TABLE contentprefer_statistic (
	id int8 NOT NULL PRIMARY KEY DEFAULT nextval('contentpreferstatistic_id_seq'::regclass),
	contenttype varchar,
	content varchar,
	pageview int4,
	updatetime timestamptz(6)
);
```




### 4.18 帮助与反馈接口

帮助与反馈接口需要sys_feedback数据表。

```sql
--创建sys_feedback表的ID序列
CREATE SEQUENCE IF NOT EXISTS sys_feedback_id_seq;
--创建sys_feedback表
CREATE TABLE IF NOT EXISTS sys_feedback (
  id int4 NOT NULL PRIMARY KEY DEFAULT nextval('sys_feedback_id_seq'::regclass),
  type int4,
  question text,
  answer varchar(255),
  status int4,
  creator int8,
  createtime timestamp(0),
  updatetime timestamp(0),
  page varchar(150),
  platform int8
);
```



### 4.19 业务事实通用接口

业务事实通用接口将业务逻辑保存在数据表中，通过业务事实通用接口进行图表数据的查询与生成。

业务事实通用接口需要bizsql数据表，按不同项目的schema进行存放，假如某项目的schema为XXXXX，则这个项目的bizsql数据表创建语句为：

```sql
CREATE TABLE IF NOT EXISTS XXXXX.bizsql (
	bizname varchar(255) NOT NULL PRIMARY KEY,
	displayname varchar(255),
	description varchar,
	tablename varchar,
	expression varchar,
	condition varchar,
	groupfield varchar,
	orderfield varchar
);
```



### 4.20 手账平台

手账平台为独立的子平台，数据表按不同项目分Schema存储，以下内容假设某项目手账平台的Schema名为xxxxx。

```sql
--创建项目Schema
CREATE SCHEMA XXXXX;
--创建zyy_resource表的ID序列
CREATE SEQUENCE IF NOT EXISTS XXXXX.zyy_resource_id_seq;
--创建zyy_resource表
CREATE TABLE IF NOT EXISTS XXXXX.zyy_resource (
	id int4 NOT NULL PRIMARY KEY DEFAULT nextval('XXXXX.zyy_resource_id_seq'::regclass),
	name varchar(32),
	displayname varchar(32),
	description varchar(255),
	type int4,
	icon varchar(32),
	background int4,
	widget int8,
	actionurl varchar(255),
	weight int4,
	recommend int4,
	creator int4,
	createtime timestamp(0),
	updator int4,
	updatetime timestamp(0),
	isdeleted int4,
	status int4 DEFAULT 1,
	isauthorize bool DEFAULT false
);
--创建zyy_tag表
CREATE TABLE IF NOT EXISTS XXXXX.zyy_tag (
	id int8,
	name varchar(255),
	type int2,
	count int4,
	weight int8
);
--创建zyy_resource_tag表
CREATE TABLE IF NOT EXISTS XXXXX.zyy_resource_tag (
	id int8,
	resourceid int8,
	tagid int8,
	creator varchar(255),
	createtime timestamp(6)
);
--创建zyy_workspace表的ID序列
CREATE SEQUENCE IF NOT EXISTS XXXXX.zyy_workspace_id_seq;
--创建zyy_workspace表
CREATE TABLE IF NOT EXISTS XXXXX.zyy_workspace (
	id int8 NOT NULL PRIMARY KEY DEFAULT nextval('XXXXX.zyy_workspace_id_seq'::regclass),
	userid int8,
	title varchar(255),
	contenttype int2,
	content varchar(255),
	actionurl varchar(255),
	creator int8,
	weight int4,
	creattime timestamp(6)
);
--创建资源内容类型表contenttype
CREATE TABLE IF NOT EXISTS XXXXX.contenttype (
	name varchar(255) NOT NULL PRIMARY KEY,
	tablename varchar(255),
	idfield varchar(255),
	filtercondition varchar(255),
	displayname varchar(255),
	description varchar(255),
	sindex int4,
	canlike bool DEFAULT false,
	cancollect bool DEFAULT false,
	canfollow bool DEFAULT false,
	canshare bool DEFAULT false,
	cancomment bool DEFAULT false
);
--插入zyy_resource默认内置数据
INSERT INTO XXXXX.zyy_resource VALUES (1, '本底信息', '基本信息', NULL, 0, 'icon_ztfx', NULL, 853, '/layout/topicview/872?districtId=442000&title=基本信息', 8, 1, NULL, '2021-05-12 17:37:37', NULL, NULL, 0, 2, 't');
--插入zyy_tag默认内置数据
INSERT INTO XXXXX.zyy_tag VALUES (1, '综合', 1, 0, 8);
--插入zyy_resource_tag默认内置数据
INSERT INTO XXXXX.zyy_resource_tag VALUES (1, 1, 1, NULL, NULL);
--插入zyy_workspace默认内置数据
INSERT INTO XXXXX.zyy_workspace VALUES (1, 0, '重点项目', NULL, '{"id":1151}', NULL, 10009, 109, '2021-10-29 16:56:59.044');
--插入contenttype默认内置数据
INSERT INTO XXXXX.contenttype VALUES ('XXXXX.zyy_resource', 'XXXXX.zyy_resource', 'id', NULL, '资源', '手账平台所有资源', 0, 't', 't', 't', 't', 't');
INSERT INTO XXXXX.contenttype VALUES ('XXXXX.zyy_resource0', 'XXXXX.zyy_resource', 'id', 'type=0', '专题', '手账平台资源专题数据', 1, 't', 't', 't', 't', 't');
INSERT INTO XXXXX.contenttype VALUES ('XXXXX.zyy_resource2', 'XXXXX.zyy_resource', 'id', 'type=2', '分局', '手账平台资源分局数据', 2, 't', 't', 't', 't', 't');
```




## 5. 数据表更新维护日志
### 2021年8月
#### 2021-08-26

1. graphmeta表的creator字段从int4类型调整成int8类型。

### 2021年9月
#### 2021-09-15

1. 增加帮助与反馈接口相关数据表（sys_feedback）。

#### 2021-09-17
1. 增加用户消息接口相关数据表（usermessage、usermessage_status）。

### 2021年12月
#### 2021-12-24
1. 增加评论点赞接口相关数据表（usercomment、contentprefer_statistic、contenttype）。

### 2022年1月
#### 2022-01-06
1. 增加专题目录接口相关数据表（topiccatalog_detail）。

#### 2022-01-11
1. mapserver表的moreinfo字段从varchar(255)改成varchar。

#### 2022-01-13
1. contenttype表增加canlike、cancollect、canfollow、canshare、cancomment等权限控制字段。

#### 2022-01-24
1. dbtable_field表增加unit字段。

2. 增加d_unit_type单位类型维度表。

3. 增加convertunit数值单位转换方法。

### 2022年2月
#### 2022-02-15
1. 增加手账平台相关数据表（zyy_resource、zyy_tag、zyy_resource_tag、zyy_workspace）。

#### 2022-02-18

1. layer表增加displaylevel字段。

### 2022年3月
#### 2022-03-03
1. filemeta表增加文件扩展信息extinfo字段。

#### 2022-03-22
1. graphmeta表增加datasource2字段。

#### 2022-03-31
1. 增加业务事实通用接口相关数据表（bizsql）。

### 2022年4月
#### 2022-04-20
1. graphmeta表增加updatetime、updator、interaction字段。

2. graphtemplate表增加interaction、formattype字段。

#### 2022-04-29
1. graphmeta表删除datasource2字段。

### 2022年5月
#### 2022-05-20
1. 更新单位类型维度表（d_unit_type）内容。

2. 更新数值单位转换方法（convertunit）。

### 2022年6月
#### 2022-06-09
1. operatorcatalog表增加isauthorize布尔型字段，默认值false，SQL算子的isauthorize字段值为true。

2. userrights表增加resourcetype='operatorcatalog'的权限设置条目。

#### 2022-06-16
1. 模型结果表modelresult增加updatetime字段。

2. 模型结果表modelresult增加唯一索引modelresult_unique_cons。

### 2022年7月
#### 2022-07-06
1. 模型分析表modelanalysis增加定时任务jobid、cronspec字段。

### 2022年8月
#### 2022-08-15
1. 用户权限表userrights增加id字段的自增序列。

2. 用户权限表userrights增加userorroleid, resourcetype, resourceid的唯一索引。

### 2022年9月
#### 2022-09-01
1. 模型定义详情表modeldetail增加模型编号modelid字段。

#### 2022-09-06
1. 专题配置表topicmeta增加authmode字段。

2. 模型目录表modelcatalog增加authmode字段。
