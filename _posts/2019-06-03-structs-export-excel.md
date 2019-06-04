---
layout:     post
title:      "Struts2 导出 Excel 报表"
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


## Struts2 导出 Excel 报表

本文内容为
- `Html button` 绑定按钮事件
- `downLoadIframe`调取下载接口
- `Struts2 Action` 业务调用及封装数据，构造 Excel 报表


#### html

按钮事件
```html
<input type="button" class="button" value="导出"  onclick="exportExcel()" />
```

#### js

使用`downLoadIframe`调用接口下载Excel文档

> `iframe`标签会创建包含另外一个文档的内联框架，跟当前`window`框架是父子关系，利用`iframe`，我们可以处理异步无刷新上传、下载文件

```js
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
			ExcelUtil.exportExcel2007(wb, sheetName, columnNames, allRows, out);
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

#### BaseAction

`response`设置`Header`及`Content-Type`
```java

public void renderExportFile(String fileName, String contentType) throws Exception {
	HttpServletResponse response = getResponse();
	response.setContentType(contentType);
	response.setHeader("Content-Disposition", "attachment;filename=" + new String(fileName.getBytes("utf-8"), "iso8859-1") + "");
	response.setHeader("Cache-Control", "private");
	response.flushBuffer();
}
```

#### Util

`Excel`导出工具类
```java
public static OutputStream exportExcel2007(Workbook wb, String sheetName, List<String> columnNames, List<List<String>> allRows, OutputStream out) throws Exception{
	CreationHelper createHelper = wb.getCreationHelper();
	Sheet sheet = wb.createSheet(sheetName);
	short index = 0;
	while(index < columnNames.size()) {
		sheet.setColumnWidth(index, 6500);
		index++;
	}
	Row row;
	Cell cell;
	row = sheet.createRow(0);
	for(int j = 0; j < columnNames.size(); j ++){
		cell = row.createCell(j);
		cell.setCellValue(createHelper.createRichTextString(columnNames.get(j)));
	}
	for(int i = 1; i <= allRows.size(); i ++){
		row = sheet.createRow(i);
		List<String> rowData = allRows.get(i - 1);
		for(int j = 0; j < rowData.size(); j ++){
			cell = row.createCell(j);
			String value = rowData.get(j);
			cell.setCellValue(createHelper.createRichTextString(value));
		}
	}
	return out;
}
```