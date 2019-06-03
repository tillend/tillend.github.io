---
layout:     post
title:      "struts2导出Excel"
subtitle:   ""
date:       2019-06-03 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Struts2
    - JavaScript
    - Excel
---


## struts2导出Excel

#### html

按钮事件
```html
<input type="button" class="button" value="导出"  onclick="exportExcel()" />
```

#### js

使用`downLoadIframe`调用接口下载Excel文档

> `iframe`标签会创建包含另外一个文档的内联框架，跟当前`window`框架是父子关系，利用`iframe`，我们可以处理异步无刷新上传、下载文件

```javascripts
<script type="text/javascript">
function exportExcel(){
	var params = getConParams();
	if(confirm("确定导出？")){
		var url = 'exportContractList.action?' + params;
		var ifr;
		if (document.getElementById('downLoadIframe') == null) {
			ifr = document.createElement("iframe");
			ifr.id = "downLoadIframe";
			ifr.style.display = "none";
			document.body.appendChild(ifr);
		}
		else {
			ifr = document.getElementById('downLoadIframe');
		}
		ifr.src = url;
	}
}
</script>
```

#### Action

- 调取`service`，获取数据
- 构造Excel
	1. 定义列名
	2. 封装数据

```java
public class UserAction extends ManagerBaseAction{

	public void exportList() throws Exception{
		String name = this.getRequest().getParameter("name");
		String gender = this.getRequest().getParameter("gender");

		Property values = new Property();
		values.put("name", actname);
		values.put("gender", gender);

		json = userService.listItem(values);

		List<Property> list = (List<Property>) JSONArray.toCollection(json.getJSONArray("list"), Property.class);

		exportExcel(list);
	}

	private void exportExcel(List<Property> list) throws Exception{
		String sheetName = "sheet";
		String excelName = sheetName + ".xlsx";

		// 列名
		List<String> columnNames = getColumnNames();

		// 结果封装
		List<List<String>> allRows = getAllRows(list);

		OutputStream out = getResponse().getOutputStream();
		renderExportFile(excelName, "application/vnd.ms-excel");

		try {
			Workbook wb = new SXSSFWorkbook();
			ExcelUtil.exportExcel2007new(wb, sheetName, columnNames, allRows, out);
			wb.write(out);
			out.flush();
		} catch (Exception e) {
			logger.warn("exportExcel2007 error.", e);
		} finally {
			out.close();
		}
	}

	private List<String> getColumnNames() {
		List<String> columnNames = new ArrayList<>();
		columnNames.add("名称");
		columnNames.add("性别");
		columnNames.add("昵称");
		columnNames.add("状态");

		return columnNames;
	}

	private List<List<String>> getAllRows(List<Property> list) {
		List<List<String>> resList = new ArrayList<>();
		for(Property p:list){
			List<String> rows = new ArrayList<>();
			rows.add(p.get("name"));
			rows.add(p.get("gender"));
			rows.add(p.get("nick"));
			rows.add(p.get("status"));

			resList.add(rows);
		}
		return resList;
	}
}
```
