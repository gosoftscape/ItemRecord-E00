# ItemRecord-E00
## 特殊业务逻辑
1. 客户学习计划.
  - 每个客户可以有多个学习计划,数量不限.
  - 学习计划的数据结构是一张主表搭配五张阶段表,主表记录学习计划的状态,五张阶段表依次记录学习计划的基本信息,申请,offer,录取证明,签证.
  - 一个学习计划有一个课程,此外可能试具体情况有多个后续课程,每个课程单独填写中介佣金,页面上在offer阶段需要有动态添加表单用于填写后续课程信息,数据上为此单独创建一张offer后续课程表以体现一对多的关系.
  - 学习计划的状态有九个,分别为"基本信息已录入","申请已递交","申请回执已接收","申请被拒绝","放弃申请","offer已接收","放弃offer","录取证明已录入","签证已录入".
  - 学习计划的状态分别影响哪些阶段可以处于激活状态被用户填写,每个阶段下哪些按钮可以处于激活状态被用户点击.
  - 添加学习计划的前提是填写基本信息,后端负责在学习计划表中添加一条记录,初始状态为"基本信息已录入",在学习计划基本信息表中添加一条记录,关联学习计划表并记录基本信息,在另外四张阶段表中分别添加一条记录,关联学习计划表,其他数据为"NULL",此后,所有学习计划的五个阶段都是做数据库的更新操作.
  - 目录结构/url/权限的组织跟随页面的结构组织(其实也是跟随数据结构组织),主要是客户->学习计划->阶段的关系.
  - 每个客户有一个学习计划状态,将这个客户的所有学习计划的状态中,最领先的(不是时间上最新)状态作为这个客户的学习计划状态.
2. 财务管理:将每一个学习计划中的课程中介佣金重复地记录在财务信息表中,包括客户信息,学习计划信息,课程信息,课程中介佣金,结算状态.
3. 数据权限和功能权限:每个中介角色的用户进入系统只能看到属于自己中介的客户(数据权限),每个学习阶段的不同步骤由不同的中介角色填写(功能权限).
4. 字典定制.
  - 使用CustomizeDictionary表保存对象类型和对象字典,对象字典结构如下:
    ```
    [{
        $fieldKey:{
            name:$fieldName,
            type:[text|number|select|multiSelect|cascade|file|date|time]
            option:$fieldOption,
            isRequired:[true|false],
            status:[true|false]
        },
        ...
    }]
    ```
    - fieldKey使用两位小写字符,从aa依次递增至zz,每次遍历所有fieldKey按字母表顺序递增,获取下一个fieldKey,同一个对象总共可以有26*26=676个字段,不考虑超出这个数字的情况.
    - name存储字段名称.
    - type存储字段类型,可选类型包括:

      类型|作用|数据库类型
      ----|----|----------
      text($length)|文本|varchar(n)
      number|数字|int
      select|单选|varchar(2)
      multiSelect|多选|varchar(2)
      cascade|级联|varchar(2)
      file|文件|varchar(255)/varchar(255)
      date|日期|varchar(8)
      time|时间|varchar(6)

      - 文本类型需要有长度选择框,长度支持修改.
      - 数值类型统一乘以100,避免使用小数.
      - 单选/多选/级联类型key使用两位小写字符,从aa依次递增至zz,每次遍历所有key按字母表顺序递增,获取下一个key.
      - 文件类型使用两个字段分别叫$fieldKey+FileURL,$fieldKey+FileName存储文件URL和文件名称.
      - 日期/时间类型使用公司规范.
    - option:当类型为单选多选级联三种选择时,此字段用于存储备选值
      - 单选多选时,此字段结构如下:
      ```
      [
        {
          key:$key,
          value:$value,
        },
        ...
      ]
      ```
      - 级联时,此字段结构如下:
      ```
      [
        {
          key:$key,
          value:$value,
          children:[{ //可选
            ...
          }]
        },
        ...
      ]
      ```
    - isRequired存储该字段是否必填,用布尔值表示.
    - status存储该字段是否启用,用布尔值表示.
  - 使用CustomizeCustomerInfo表存储客户信息(暂时保留CustomerInfo表),该表初始状态只有id字段,作为主键,随后根据字典定制功能中的定制,动态添加/删除/修改/移动字段.
  - 字典定制功能左边栏选择对象,目前只支持定制客户对象,右边是字段列表,有字段添加/删除/修改/移动/启用/禁用功能,选项添加/删除/修改/移动功能.
    - 移动就是使用拖拽来改变选中字段的顺序,使用sql的修改字段在另一个字段后来实现.
    - 修改只支持修改除类型之外的字段属性,和文本类型的长度.
    - 禁用意味着不删除字段,保留字段数据的情况下,达到删除字段的其他效果,启用是其相反效果.
  - 字典定制前后端交互使用前端记录定制行为,点击保存按钮时,传递指令序列,后端提供POST方法,按照指令序列对对象表表结构进行修改的方式,指令格式如下.
    ```
    [
      {
        action:fieldInsert,
        name,
        type,
        isRequired,
        status,
      },
      {
        action:fieldDelete,
        fieldKey,
      },
      {
        action:fieldUpdate,
        key:[name|type|value|isRequired|status],
        value,
      },
      {
        action:fieldUpdateSort,
        srcFieldKey, //被移动的字段
        dstFieldKey, //移动到该字段之后
      },
      {
        action:[fieldEnable|fieldDisable],
        fieldKey,
      },
      {
        action:optionInsert,
        fieldKey,
      },
      {
        action:optionDelete,
        fieldKey,
        optionKey,
      },
      {
        action:optionUpdate,
        fieldKey,
        optionKey,
        optionValue,
      },
      {
        action:optionUpdateSort,
        fieldKey,
        srcOptionKey, //被移动的选项
        dstOptionKey, //移动到该选项之后
      },
    ]
    ```
  - 后端提供按照对象名称查询该对象字典的功能,用于前端进入客户管理页面之后查询,字典数据结合客户数据共同驱动页面,这个部分由于前端需要处理很多逻辑,所以把禁用字段的功能也放在前端一起处理.
  - 客户管理详情页面,暂时不加区别地使用一行一列的格式显示.
  - 根据条件查询客户的功能,后端由于不知道字段,无法使用惯用的mybatis动态sql功能,这部分需要使用jdbcTemplate遍历参数动态处理条件/分页/排序的功能.
  - 创建一个git分支实现字典定制功能,由于这部分功能实现有可能耽误线上版本的小幅度修改需求,所以需要使用分支机制.

## 问题记录
### 开发阶段

### 测试阶段

### 维护阶段