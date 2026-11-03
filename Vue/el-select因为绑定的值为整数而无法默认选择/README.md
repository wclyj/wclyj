前言
--

字典项的值在表单里是整数，下拉框却只显示那个数字，不高亮对应选项。字符串字段没有这个问题，一换成整数就出现。

```html
<el-form-item label="层级" :label-width="labelWidth" required>
  <el-select v-model="form.rank" size="small" placeholder="请选择">
    <el-option
      v-for="item in rankOptions"
      :key="item.id"
      :label="item.label"
      :value="item.code"
    />
  </el-select>
</el-form-item>
```

`el-option` 用严格相等比较当前值和选项值。接口把字典码放在字符串里，`v-model` 却是数字，`"2" === 2` 不成立，于是框里回显原始数字，而不是选项文案。

在选项上把字典码收成整数，两边类型一致后就能对上：

```html
<el-form-item label="层级" :label-width="labelWidth" required>
  <el-select v-model="form.rank" size="small" placeholder="请选择">
    <el-option
      v-for="item in rankOptions"
      :key="item.id"
      :label="item.label"
      :value="parseInt(item.code, 10)"
    />
  </el-select>
</el-form-item>
```

也可以反过来，提交前再把表单值转成数字，选项保持字符串。不要混用。`parseInt` 遇到空串会得到 `NaN`，空选项要提前滤掉，否则会出现一个选不中的空项。
