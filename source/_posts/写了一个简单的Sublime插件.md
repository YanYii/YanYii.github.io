---
title: 写了一个简单的Sublime插件
tags: 敏捷开发
date: 2020-11-24
excerpt: JSON生成Java实体类。记录下简单、直接的快速迭代实现思路
---

动手写一个简单的Sublime插件，实现将JSON字符串生成Java实体类

最近工作上经常对接第三方接口，需要按照接口文档，编写入参、出参对应的实体类。每个类的成员，少则几个，多则十多二十个。写一两个还好，写上四五个，明显是在重复劳动。

要是可以把文档里给的json示例，自动变成实体类，不用一个一个写就好了。

茫茫互联网，我不是第一个有这种想法的人。百度一下，找到一些在线工具，支持json字符串转成Java实体类。转换出来的类，稍作修改也可以用。奈何公司开发是在内网环境，没法上百度。刚好自己会写点Python和Sublime插件。就试试用Sublime插件，写一个Sublime插件，来实现它吧。

<b>实现完的操作效果大致如下：</b>
1.将接口文档中的json串手动复制粘贴到Sublime
2.一键执行自己写的插件，在当前页面直接输出生成的实体类
3.将实体类复制到IDEA中，稍作调整，继续愉快的开发

![](/images/sublime_java_entity.gif)

<b>插件实现的思路大致如下：</b>
1.读取当前页面的全部内容，即待生成实体类的json串，转换为Python的dict数据
2.通过dict数据构造出实体类字符串
3.将字符串打印输出到当前页面

构造实体类字符串的思路大致如下：
1.实体类有分为三部分，（get，set方法不生成）。

	- 头部分为 public class Data {，头部的实现先写死
	- 成员部分为 private String key; 成员部分的key由json的参数决定，遍历dict可得所有的key
	- 尾部分为 } ，尾巴部分写死

2.如果json里的key，对应的value，还是一个json对象，需要继续转换为实体类。

	- 用递归实现。
	- 注意子级实体类字符串的位置，这里规定下，在当前类的全部成员输出完后，再依次输出子级实体类
	- 递归 + 依次输出，类似多子树的深度优先前序遍历。

3.完成以上全部功能后，大致的框架结构就出来了。看看执行效果，继续完善细节

4.完善成员的类型：

	- 成员部分的key，对应的类型不全是String，需要写一个转换函数，由value去判断具体的类型。基本类型可以完成，复杂类型，暂时略过写固定为Object。

5.完善格式，indent缩进，以及缩进的层级。

	- 缩进的层级与递归的层级也有关，将indent=0，作为参数传入递归函数
	- ‘  ’ * indent  Python支持这种字符串乘法语义，快速计算出缩进的空格数

6.List类型的处理：

	- 完善value为list时，key在Java中的类型为List<Object>，这里有一种是List<String>，一种是List<Object>
	- 完善生成的实体类名称，定义规则：第一个用Data，其他的用Item+缩进+在当前类的子类先后顺序，把实体类名称作为对应Key的类型
	
7.微调：

	- 微调换行
	- 补充lombok的@Data注解
	- 补充key成员名称由 is_ok 转化为驼峰规则 isOk 的处理


PS：代码框架结构出来后，其他的真的是修修补补


```python
import sublime
import sublime_plugin
import json


class GenerateJavaEntityCommand(sublime_plugin.TextCommand):

	def load_all_content(self):
		region = sublime.Region(0, self.view.size())
		content = self.view.substr(region)
		return content

	# 入口在这里
	def run(self, edit):
		# 读取文本，按json格式读取
		content = self.load_all_content()
		obj = json.loads(content)

		# 开始生成实体类，
		output = self.convert([obj], 0)
		
		# 输出结果
		self.view.insert(edit, self.view.size(), "\n" + output)

		# 将结果复制到剪贴板
		sublime.set_clipboard(output)

	def convert(self, root, indent):
		clazz = []

		for i,obj in enumerate(root):
			if type(obj) == dict:
				# 头、中间部分（包括递归的子级实体类）、尾部
				s_class_start = self.get_class_start(indent, i)
				s_class_body, leaf = self.get_body(obj, indent)
				s_class_end = self.get_class_end(indent)
				# 在这里有递归
				s_class = s_class_start + s_class_body + self.convert(leaf, indent+1) + s_class_end
				clazz.append(s_class)

		return "".join(clazz)

	def get_class_start(self, indent, idx):
		pre = "  " * (indent) + "@Data\n"
		if indent == 0:
			class_start = "  " * (indent) + "public class " +  self.get_class_name(indent, idx) + " {\n\n"
		else:
			class_start = "  " * (indent) + "public static class " +  self.get_class_name(indent, idx) + " {\n\n"
		
		return pre + class_start

	def get_body(self, obj, indent):
		body = []
		leaf = []
		for (k,v) in  obj.items():
			if type(v) == dict:
				leaf.append(v)

				s = "  " * (indent+1) +  "private {type_name} {key_name};\n".format(type_name=self.get_class_name(indent+1, len(leaf)-1), key_name=self.get_key_name(k))
			elif type(v) == list:
				if len(v) == 0:
					s = "  " * (indent+1) +  "private {type_name} {key_name};\n".format(type_name="List<Object>", key_name=self.get_key_name(k))
				else:
					if type(v[0]) == dict:
						leaf.append(v[0])
						class_name = "List<{}>".format(self.get_class_name(indent+1, len(leaf)-1))
						s = "  " * (indent+1) +  "private {type_name} {key_name};\n".format(type_name=class_name, key_name=self.get_key_name(k))
					elif type(v[0]) == str:
						s = "  " * (indent+1) +  "private {type_name} {key_name};\n".format(type_name="List<String>", key_name=self.get_key_name(k))	

			else:
				s = "  " * (indent+1) +  "private {type_name} {key_name};\n".format(type_name=self.get_type_name(v), key_name=self.get_key_name(k))
			
			body.append(s)

		body.append('\n')
		return "".join(body), leaf

	def get_class_end(self, indent):
		return "  " * (indent) + "}\n\n"

	def get_class_name(self, indent, idx):
		if indent == 0:
			return 'Data'
		else:
			return 'Item' + str(indent)+'_' + str(idx)

	def get_key_name(self, k):
		words = k.split('_')
		if len(words) == 1:
			return words[0]
		else:
			first = words[0]
			words = list(map(lambda x: x.capitalize(), words))
			words[0] = first
			return "".join(words)

	def get_type_name(self, value):
		if type(value) == str:
			return 'String'
		elif type(value) == bool:
			return 'Boolean'
		elif type(value) == int:
			return 'Integer'
		elif type(value) == float:
			return 'Float'
		else:
			return 'Object'


