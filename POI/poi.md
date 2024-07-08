### 根据存在的excel文件创建Workbook

可根据流(InputStream)或者File来创建，使用File创建内存占用更低，但是会修改文件本身，即使只是读取操作，文件的修改时间会发生变化。

```java
// 可指定第三个参数readonly为true，则不会对文件本身进行修改，但是只允许只读操作
Workbook wb = WorkbookFactory.create(new File("/Users/wangtao/Desktop/user.xlsx"), null, true);
```

### 复制或者读取校验规则

```java
/**
 * 复制sheet中校验规则, 目前只支持下拉框校验
 * @param srcSheet 原始sheet
 * @param destSheet 目标sheet
 */
public static void copyValidation(Sheet srcSheet, Sheet destSheet) {
    List<? extends DataValidation> dataValidations = srcSheet.getDataValidations();
    for (DataValidation dataValidation : dataValidations) {
        // 复制校验规则
        destSheet.addValidationData(dataValidation);

        DataValidationConstraint validationConstraint = dataValidation.getValidationConstraint();
        // 下拉框
        if (DataValidationConstraint.ValidationType.LIST == validationConstraint.getValidationType()) {
            /*
             * 如果下拉框内容是引用别的单元格里的内容，则需要复制到目标sheet中
             * 两种情况:
             * 校验规则中直接指定枚举值, validationConstraint.getExplicitListValues()可以获取到
             * 结果: [男, 女]
             * 引用单元格区域(必须是一列或者一行), validationConstraint.getFormula1()可以获取到
             * 结果: $C$2:$C$3, 引用第C列第2行-第3行两个单元格
             */
            if (validationConstraint.getExplicitListValues() == null && validationConstraint.getFormula1() != null) {
                // 构造区域
                AreaReference areaReference = new AreaReference(validationConstraint.getFormula1(),
                        srcSheet.getWorkbook().getSpreadsheetVersion());
                CellReference[] allReferencedCells = areaReference.getAllReferencedCells();
                DataFormatter dataFormatter = new DataFormatter();
                for (CellReference cellReference : allReferencedCells) {
                    if (cellReference.getSheetName() == null) {
                        Row srcRow = srcSheet.getRow(cellReference.getRow());
                        Cell srcCell = srcRow.getCell(cellReference.getCol());
                        String enumValue = dataFormatter.formatCellValue(srcCell);

                        Row destRow = getOrCreateRow(destSheet, cellReference.getRow());
                        Cell destCell = destRow.createCell(cellReference.getCol());
                        destCell.setCellValue(enumValue);
                    } else {
                        // TODO: 处理引用其他sheet的情况, 暂不考虑
                    }
                }
            }
        }
    }
}

public static Row getOrCreateRow(Sheet sheet, int rowNum) {
    Row row = sheet.getRow(rowNum);
    if (row == null) {
        row = sheet.createRow(rowNum);
    }
    return row;
}
```

