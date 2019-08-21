### 前言 ###
一直在用tk.mybatis也没深入的看过源码，今天看源码的原因是因为在entity中加了个字段，结果这个字段被用来生成sql，然而并不想让这个字段也拼到sql中去，才有了今天的这份记录。<b>解决方案在文末。</b><br/>
tk.mybatis作用是能够为所有的mapper自动添加增删改查方法，继承Mapper<T>就行。
### 实现 ###
系统启动时，tk.mybatis会向mybatis注册所有自动生成的接口，MappedStatement也会放进去，之后会加载这些接口，为每个MappedStatement设置对应的sqlSource（如果是我们自定义的sql，则这个sqlSource就是我们xml里面的sql，如果是自动生成的接口就是待会拼接出来的sql）<br/>
例如：根据主键id查询entity方法，sql拼接的源码
```
public String selectByPrimaryKey(MappedStatement ms) {
    //这段代码最关键
    Class<?> entityClass = this.getEntityClass(ms);
    this.setResultType(ms, entityClass);
    StringBuilder sql = new StringBuilder();
    sql.append(SqlHelper.selectAllColumns(entityClass));
    sql.append(SqlHelper.fromTable(entityClass, this.tableName(entityClass)));
    sql.append(SqlHelper.wherePKColumns(entityClass));
    return sql.toString();
}
```
关键代码在getEntityClass（）方法，tk.mybatis会将entity对应的tableName以及底下的columnName缓存下来，还涉及到了缓存字段的一些规则。实际是缓存到EntityTable中，之后该entity生成其余sql的时候，直接取出来拼接即可。
```
/**
 * 获取返回值类型 - 实体类型
 *
 * @param ms
 * @return
     */
public Class<?> getEntityClass(MappedStatement ms) {
    String msId = ms.getId();
    if (entityClassMap.containsKey(msId)) {
        return entityClassMap.get(msId);
    } else {
        Class<?> mapperClass = getMapperClass(msId);
        Type[] types = mapperClass.getGenericInterfaces();
        for (Type type : types) {
            if (type instanceof ParameterizedType) {
                ParameterizedType t = (ParameterizedType) type;
                if (t.getRawType() == this.mapperClass || this.mapperClass.isAssignableFrom((Class<?>) t.getRawType())) {
                    Class<?> returnType = (Class<?>) t.getActualTypeArguments()[0];
                    //这段代码是最关键的，加载entity的基本信息（entity对应的tableName，以及entity底下的字段对应表的字段）
                    //生成基本的增删改查sql都要用到这些信息
                    //同时entity底下的字段还有一定的加载规则，并不是entity底下的所有字段都会用来拼接。
                    EntityHelper.initEntityNameMap(returnType, mapperHelper.getConfig());
                    entityClassMap.put(msId, returnType);
                    return returnType;
                }
            }
        }
    }
    throw new MapperException("无法获取 " + msId + " 方法的泛型信息!");
}
```
底下是EntityHelper.initEntityNameMap(returnType, mapperHelper.getConfig());源码
```
/**
 * 初始化实体属性
 *
 * @param entityClass
 * @param config
 */
public static synchronized void initEntityNameMap(Class<?> entityClass, Config config) {
    if (entityTableMap.get(entityClass) != null) {
        return;
    }
    //创建并缓存EntityTable
    EntityTable entityTable = resolve.resolveEntity(entityClass, config);
    entityTableMap.put(entityClass, entityTable);
}
```
resolve.resolveEntity(entityClass, config);源码
```
@Override
public EntityTable resolveEntity(Class<?> entityClass, Config config) {
    Style style = config.getStyle();
    //style，该注解优先于全局配置
    if (entityClass.isAnnotationPresent(NameStyle.class)) {
        NameStyle nameStyle = entityClass.getAnnotation(NameStyle.class);
        style = nameStyle.value();
    }

    //创建并缓存EntityTable
    EntityTable entityTable = null;
    if (entityClass.isAnnotationPresent(Table.class)) {
        Table table = entityClass.getAnnotation(Table.class);
        if (!"".equals(table.name())) {
            entityTable = new EntityTable(entityClass);
            entityTable.setTable(table);
        }
    }
    if (entityTable == null) {
        entityTable = new EntityTable(entityClass);
        //可以通过stye控制
        String tableName = StringUtil.convertByStyle(entityClass.getSimpleName(), style);
        //自动处理关键字
        if (StringUtil.isNotEmpty(config.getWrapKeyword()) && SqlReservedWords.containsWord(tableName)) {
            tableName = MessageFormat.format(config.getWrapKeyword(), tableName);
        }
        entityTable.setName(tableName);
    }
    entityTable.setEntityClassColumns(new LinkedHashSet<EntityColumn>());
    entityTable.setEntityClassPKColumns(new LinkedHashSet<EntityColumn>());
    //处理所有列
    List<EntityField> fields = null;
    if (config.isEnableMethodAnnotation()) {
        fields = FieldHelper.getAll(entityClass);
    } else {
        fields = FieldHelper.getFields(entityClass);
    }
    for (EntityField field : fields) {
        //如果启用了简单类型，就做简单类型校验，如果不是简单类型，直接跳过
        //3.5.0 如果启用了枚举作为简单类型，就不会自动忽略枚举类型
        //4.0 如果标记了 Column 或 ColumnType 注解，也不忽略
        if (config.isUseSimpleType()
                && !field.isAnnotationPresent(Column.class)
                && !field.isAnnotationPresent(ColumnType.class)
                &&
                //如果字段类型是简单类型，也不会忽略
                !(SimpleTypeUtil.isSimpleType(field.getJavaType())
                ||
                (config.isEnumAsSimpleType() && Enum.class.isAssignableFrom(field.getJavaType())))) {
            continue;
        }
        processField(entityTable, field, config, style);
    }
    //当pk.size=0的时候使用所有列作为主键
    if (entityTable.getEntityClassPKColumns().size() == 0) {
        entityTable.setEntityClassPKColumns(entityTable.getEntityClassColumns());
    }
    entityTable.initPropertyMap();
    return entityTable;
}
```
processField(entityTable, field, config, style);源码
```
/**
 * 处理字段
 *
 * @param entityTable
 * @param field
 * @param config
 * @param style
 */
protected void processField(EntityTable entityTable, EntityField field, Config config, Style style) {
    //这里就可以看到，当我们属性不想被拼接sql的时候，在上面加个@Transient注解即可
    if (field.isAnnotationPresent(Transient.class)) {
        return;
    }
    //Id
    EntityColumn entityColumn = new EntityColumn(entityTable);
    //是否使用 {xx, javaType=xxx}
    entityColumn.setUseJavaType(config.isUseJavaType());
    //记录 field 信息，方便后续扩展使用
    entityColumn.setEntityField(field);
    if (field.isAnnotationPresent(Id.class)) {
        entityColumn.setId(true);
    }
    //Column
    String columnName = null;
    if (field.isAnnotationPresent(Column.class)) {
        Column column = field.getAnnotation(Column.class);
        columnName = column.name();
        entityColumn.setUpdatable(column.updatable());
        entityColumn.setInsertable(column.insertable());
    }
    //ColumnType
    if (field.isAnnotationPresent(ColumnType.class)) {
        ColumnType columnType = field.getAnnotation(ColumnType.class);
        //是否为 blob 字段
        entityColumn.setBlob(columnType.isBlob());
        //column可以起到别名的作用
        if (StringUtil.isEmpty(columnName) && StringUtil.isNotEmpty(columnType.column())) {
            columnName = columnType.column();
        }
        if (columnType.jdbcType() != JdbcType.UNDEFINED) {
            entityColumn.setJdbcType(columnType.jdbcType());
        }
        if (columnType.typeHandler() != UnknownTypeHandler.class) {
            entityColumn.setTypeHandler(columnType.typeHandler());
        }
    }
    //列名
    if (StringUtil.isEmpty(columnName)) {
        columnName = StringUtil.convertByStyle(field.getName(), style);
    }
    //自动处理关键字
    if (StringUtil.isNotEmpty(config.getWrapKeyword()) && SqlReservedWords.containsWord(columnName)) {
        columnName = MessageFormat.format(config.getWrapKeyword(), columnName);
    }
    entityColumn.setProperty(field.getName());
    entityColumn.setColumn(columnName);
    entityColumn.setJavaType(field.getJavaType());
    if (field.getJavaType().isPrimitive()) {
        log.warn("通用 Mapper 警告信息: <[" + entityColumn + "]> 使用了基本类型，基本类型在动态 SQL 中由于存在默认值，因此任何时候都不等于 null，建议修改基本类型为对应的包装类型!");
    }
    //OrderBy
    processOrderBy(entityTable, field, entityColumn);
    //处理主键策略
    processKeyGenerator(entityTable, field, entityColumn);
    //将该字段维护到entityTable中
    entityTable.getEntityClassColumns().add(entityColumn);
    if (entityColumn.isId()) {
        entityTable.getEntityClassPKColumns().add(entityColumn);
    }
}
```
附一份tk.mybatis维护的简单类型
```
/**
 * 特别注意：由于基本类型有默认值，因此在实体类中不建议使用基本类型作为数据库字段类型
 */
static {
    SIMPLE_TYPE_SET.add(byte[].class);
    SIMPLE_TYPE_SET.add(String.class);
    SIMPLE_TYPE_SET.add(Byte.class);
    SIMPLE_TYPE_SET.add(Short.class);
    SIMPLE_TYPE_SET.add(Character.class);
    SIMPLE_TYPE_SET.add(Integer.class);
    SIMPLE_TYPE_SET.add(Long.class);
    SIMPLE_TYPE_SET.add(Float.class);
    SIMPLE_TYPE_SET.add(Double.class);
    SIMPLE_TYPE_SET.add(Boolean.class);
    SIMPLE_TYPE_SET.add(Date.class);
    SIMPLE_TYPE_SET.add(Timestamp.class);
    SIMPLE_TYPE_SET.add(Class.class);
    SIMPLE_TYPE_SET.add(BigInteger.class);
    SIMPLE_TYPE_SET.add(BigDecimal.class);
    //反射方式设置 java8 中的日期类型
    for (String time : JAVA8_DATE_TIME) {
        registerSimpleTypeSilence(time);
    }
}
```
### 总结 ###
终究解决方案直接在属性上加@Transient注解，包路径：javax.persistence。