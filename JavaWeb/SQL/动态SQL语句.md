# 动态SQL语句的一些标签
### 基础：
动态SQL语句是使用在**xml映射文件**中的<br>
xml映射文件的名字应该要**与类名相同**并且要在相**同路径名称**下<br>
> 例如在项目中
>> main / java / com / example / mapper / EmpExprMapper.java
>> main / resources / com / example / mapper / EmpExprMapper.xml


## ① where 和 if 标签
* where标签用于将条件语句拼接成WHERE子句
    * id 属性：指定方法名
    * resultType 属性：指定返回的结果类型
```xml
    <select id="list" resultType="com.example.pojo.Emp">
        SELECT e.*, d.name AS deptName
        FROM emp e LEFT JOIN dept d ON e.dept_id = d.id
        <where>
            <if test="name != null and name != ''">
                AND e.name LIKE CONCAT('%', #{name}, '%')
            </if>
            <if test="gender != null">
                AND e.gender = #{gender}
            </if>
            <if test="begin != null">
                AND e.entry_date >= #{begin, jdbcType=DATE}
            </if>
            <if test="end != null">
                AND e.entry_date <![CDATA[ <= ]]> #{end, jdbcType=DATE}
            </if>
        </where>
        ORDER BY e.update_time DESC
    </select>
```

## ② foreach标签
* foreach标签属性值：
    * collection 属性：指定集合名称
    * item ：指定集合中的元素
    * separator ：指定分隔符
    * open : 遍历开始前的拼接片段
    * close : 遍历结束后的拼接片段
```xml
<insert id="insertBatch">
    insert into emp_expr(emp_id, begin, end, company, job) values
    <foreach collection="exprList" item="expr" separator=",">
        (#{expr.empId},#{expr.begin},#{expr.end},#{expr.company},#{expr.job})
    </foreach>
</insert>
```

## ③ set标签
* 会自动生成set关键字，会自动删除更新字多后多余','
```xml
<update id="updateById">
    update emp
    <set>
        <if test="username != null and username != ''">username = #{username},</if>
        <if test="password != null and password != ''">password = #{password},</if>
        <if test="name != null and name != ''">name = #{name},</if>
        <if test="gender != null">gender = #{gender},</if>
        <if test="phone != null and phone != ''">phone = #{phone},</if>
        <if test="job != null">job = #{job},</if>
        <if test="salary != null">salary = #{salary},</if>
        <if test="image != null and image != ''">image = #{image},</if>
        <if test="entryDate != null">entry_date = #{entryDate},</if>
        <if test="deptId != null">dept_id = #{deptId},</if>
        <if test="updateTime != null">update_time = #{updateTime},</if>
    </set>
    where id = #{id}
</update>
```