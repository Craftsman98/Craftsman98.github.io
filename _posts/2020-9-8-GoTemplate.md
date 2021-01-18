---
tags: Go 
---

## Go Template

```yaml
# 渲染数据可以是go的所有类型
# 渲染数据本身
{{ . }}
# 自定义变量
{{ $var = . }}

# struct
# 大写字母开头属性Field 
{{ .Field }}
# 嵌套struct时的 属性Field1
{{ .Field.Field1 }}
# 变量为struct时的 属性Field
{{ $x = . }}
{{ $x.Field }}
# 方法
{{ .Method }}
# 嵌套struct的方法
{{ .Field.Method }}
# 嵌套struct的map中struct的方法
{{ .Field.Key.Method }}

# map
# 键名
{{ .key }}
# struct中map
{{ .Field.key }}
# 变量的struct中map
{{ $x = . }}
{{ $x.Field.key }}

# 函数
{{ funName . }}



# 操作
# 注释
{{/* comment */}}
{{- /* comment */ -}}

# 默认输出,效果 fmt.Print(pipeline)
{{ pipeline }}

# 流程控制
{{if pipeline}} T1 {{end}}
{{if pipeline}} T1 {{else}} T0 {{end}}
{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}
{{with pipeline}} T1 {{end}}
{{with pipeline}} T1 {{else}} T0 {{end}}

# 循环
# array, slice, map, or channel
{{range pipeline}} T1 {{end}}
{{range pipeline}} T1 {{else}} T0 {{end}}

# 嵌套关联
{{template "name"}}
# 当前模板引入其他模板,并且传递数据
{{template "name" pipeline}}
# 等价声明和执行 {{define "name"}} T1 { {end}}  & {{template "name" pipeline}}
{{block "name" pipeline}} T1 {{end}} 



# 例子
 <table border="1"> 
      <caption><h2>待处理列表</h2></caption>
      <tr><th>序号</th><th>说明</th></tr>
      # []*struct 遍历slice的struct赋值给变量
      {{ range $key, $value := .}}
      
      <tr height=40px>
             <td bgcolor="#d1d1d1" align="center">{{ $key }}</td>

          # 判断struct的属性Priority, 为Description标记不通背景颜色
          {{ if eq $value.Priority  4 }}
              <td bgcolor="#e97558">{{  $value.Description }}</td>
          {{ else if eq $value.Priority  3 }}
              <td bgcolor="#f9a059">{{  $value.Description }}</td>
          {{ else if eq $value.Priority  2 }}
             <td bgcolor="#fcc859">{{  $value.Description }}</td>
          {{ end }}
      </tr>
     {{end}}
</table>
```

