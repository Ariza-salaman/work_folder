# 笔记

## 自动生成编号

### 获取已经存在的最大编号

```sql
SELECT COALESCE(MAX(vendor_code), 0) AS code FROM vendor_base_info;
```

### 拼接字符串输出新的编号

```js
	SetNextCode: () => {
		return "GYS" + (Number(GetMaxCode.data[0].code.slice(3)) + 1).toString().padStart(3, '0');
	}
```

## 将自增表格的多条数据一次插入

### 获取自增表格数据并且转换

```js
	setItemData:() =>{
		const quotation_code = this.setNextCode(); //用于和主表单关联的字段
		const sqlValues = formily1.formData.contact_table.map(item => `('${quotation_code}','${item.contact_name}', '${item.contact_phone}', '${item.contact_position}')`).join(', ');
		return sqlValues
	},
```

### 插入多条数据

```sql
INSERT INTO vendor_contact_info (vendor_code, vendor_contact_name, vendor_contact_phone, vendor_contact_position)
  VALUES {{JSObject1.SetContactData()}};
```

## 一次更新多张表

```sql
--基础信息
INSERT INTO vendor_base_info (vendor_code, vendor_name, vendor_class, vendor_level, contract_begin_date, contract_end_date, settlement_period, credit_limit, purchase_supervisor, purchase_department, vendor_location, vendor_address, contract_file, Invoice_header, tax_code, tax_type, tax_rate, opening_bank, bank_account)
VALUES ('{{JSObject1.SetNextCode()}}', '{{formily1.formData.vendor_name}}', '{{formily1.formData.vendor_class}}', '{{formily1.formData.vendor_level}}', '{{formily1.formData.contract_begin_date}}', '{{formily1.formData.contract_end_date}}', '{{formily1.formData.settlement_period}}', '{{formily1.formData.credit_limit}}', '{{formily1.formData.purchase_supervisor}}', '{{formily1.formData.purchase_department}}', '{{formily1.formData.vendor_location}}', '{{formily1.formData.vendor_address}}', '{{formily1.formData.contract_file}}', '{{formily1.formData.Invoice_header}}', '{{formily1.formData.tax_code}}', '{{formily1.formData.tax_type}}', '{{formily1.formData.tax_rate}}', '{{formily1.formData.opening_bank}}', '{{formily1.formData.bank_account}}');
--联系信息
INSERT INTO vendor_contact_info (vendor_code, vendor_contact_name, vendor_contact_phone, vendor_contact_position)
VALUES ('{{JSObject1.SetNextCode()}}', '{{formily1.formData.contact_name}}', '{{formily1.formData.contact_phone}}', '{{formily1.formData.contact_position}}');
INSERT INTO vendor_contact_info (vendor_code, vendor_contact_name, vendor_contact_phone, vendor_contact_position)
  VALUES {{JSObject1.SetContactData()}};
--资质信息
INSERT INTO vendor_certificate_info (vendor_code, vendor__certificate_name, vendor_certificate_file)
  VALUES {{JSObject1.SetCertificateData()}};
```

## 将查询结果转换为选项

### 转换

```js
//将查询结果数字转换为选项，传入参数：myArr代表查询结果数组，n代表数组的索引号	
convertArrayToOptions:(myArr,n) =>{
const coArray = myArr.map(item => ({ label: Object.values(item)[n], value: Object.values(item)[n] }))
return  Array.from(new Set(coArray.map(JSON.stringify)), JSON.parse)
	},
//调用
//获取供应商名称为选项
	SetVendorName: () => {
		return this.convertArrayToOptions(GetVendorName.data,0)
	},
```

## 将选中项转换为查询条件

### 转换选中项

```js
	//将选项转换为查询条件，传入参数：myElement，需要转换数据的组件名称
		convertOptionsToConditions:(myElement)=>{
		const values = myElement.selectedOptionValues
		const options = myElement.options
		return values.length === 0?"'" + options.map(item => item.value).join("','") + "'":`'${values.join("','")}'`
	},
        //调用
        	SetVendorName2:()=> {
	return this.convertOptionsToConditions(MultiSelect6)
},
```

### 使用转换后的数据查询

```sql
SELECT
  a.vendor_code,
  a.standard_purchase_unit_price,
  a.purchase_unit_price,
	CONCAT((a.discount_rate * 100), '%') AS discount_rate,
  a.purchase_price_notax,
	CONCAT((a.VAT_rate * 100), '%') AS VAT_rate,
  b.product_name,
  b.product_class,
  b.product_brand,
  b.product_model,
  c.vendor_name,
  c.vendor_level
FROM
  vendor_price_info a
  left join product_info b on a.product_code = b.product_code
  left join vendor_base_info c on a.vendor_code = c.vendor_code
where
  c.vendor_name IN ({{ JSObject1.SetVendorNameOption() }})
  AND b.product_name IN ({{ JSObject1.SetProductNameOption() }})
  AND b.product_model IN ({{ JSObject1.SetProductModelOption() }})
  AND b.product_brand IN ({{ JSObject1.SetProductBrandOption() }})
```

## 数字装换为中文金额大写

```js
function numberToChinese(n) {
      var fraction = ['角', '分'];
      var digit = [
        '零', '壹', '贰', '叁', '肆',
        '伍', '陆', '柒', '捌', '玖'
      ];
      var unit = [
        ['元', '万', '亿'],
        ['', '拾', '佰', '仟']
      ];
      var head = n < 0 ? '欠' : '';
      n = Math.abs(n);
      var s = '';
      for (var i = 0; i < fraction.length; i++) {
        s += (digit[Math.floor(n * 10 * Math.pow(10, i)) % 10] + fraction[i]).replace(/零./, '');
      }
      s = s || '整';
      n = Math.floor(n);
      for (var i = 0; i < unit[0].length && n > 0; i++) {
        var p = '';
        for (var j = 0; j < unit[1].length && n > 0; j++) {
          p = digit[n % 10] + unit[1][j] + p;
          n = Math.floor(n / 10);
        }
        s = p.replace(/(零.)*零$/, '').replace(/^$/, '零') + unit[0][i] + s;
      }
      return head + s.replace(/(零.)*零元/, '元').replace(/(零.)+/g, '零').replace(/^整$/, '零元整');
    }
```

## 把外面jsObject的内容，放到里面select组件的可选项中

### 获取数据库数据

```sql
SELECT price_level FROM price_level LIMIT 10;
```

### 转换数据格式并保存到对象中

```js
// 转换格式函数
convertArrayToOptions:(myArr,n) =>{
		const coArray = myArr.map(item => ({ label: Object.values(item)[n], value: Object.values(item)[n] }))
		return  Array.from(new Set(coArray.map(JSON.stringify)), JSON.parse)
	},
调用函数并且将结果保存到对象里面
	setFormilyValue: () => {
		const selectOptions = this.convertArrayToOptions(getPriceLevel.data , 0)
		const obj = {
			"name":"selectOptions",
		}
		obj.priceLevel = selectOptions
		return obj
	},
```

###将对象作为formily表单初始值

![70365834616](C:\Users\cheng.zhou\Documents\markdown\image\1703658346160.png) 

### 配置select组件的动作响应器

![70365840151](C:\Users\cheng.zhou\Documents\markdown\image\1703658401518.png)

## formily 表单日期选择框格式设置

![70365862341](C:\Users\cheng.zhou\Documents\markdown\image\1703658623410.png)

- 如图所示，将选择器类型选为`日期`，然后打开`时间选择`开关，就可以同时先择日期和时间

- 格式`YYYY-MM-DD`表示`年-月-日` ，`HH:mm:ss`表示`时分秒` ,分和秒必须**小写** ，`HH` 表示24小时制，`hh`表示12小时制 

## formily表单字段相互关联

输入含税单价和税率，自动计算不含税单价等信息

![70373539140](C:\Users\cheng.zhou\Documents\markdown\image\1703735391407.png)

### 设置依赖字段

  ![70365916133](C:\Users\cheng.zhou\Documents\markdown\image\1703659161337.png)

### 处理逻辑

  ![70365964289](C:\Users\cheng.zhou\Documents\markdown\1703659642895.png)

  使用`$effect` 函数用于处理副作用逻辑，接受两个参数，一个箭头函数写逻辑，一个数组写依赖字段。

  ```js
  $effect(() => {
    const rate = parseInt($deps.VAT_rate.replace("%", "")) / 100
    const result = $deps.sale_price_taxin * (1 - rate) || 0
    $self.value = result.toFixed(2)
  }, [$deps.sale_price_taxin, $deps.VAT_rate])
  ```



## 根据下拉框选择结果自动填写

选择客户编号后，自动填写对应的编码，联系人等信息

![70373518305](C:\Users\cheng.zhou\Documents\markdown\image\1703735183056.png)

### 数据获取

```sql
SELECT 
a.customer_code,
a.customer_name,
b.contact_name,
b.contact_phone,
a.sale_supervisor,
a.sale_department
FROM 
customer_base_info a
LEFT JOIN
customer_contact_info b
on a.customer_code = b.customer_code
```

### 转换数据并保存到对象中

```js
// 转换格式函数
convertArrayToOptions:(myArr,n) =>{
		const coArray = myArr.map(item => ({ label: Object.values(item)[n], value: Object.values(item)[n] }))
		return  Array.from(new Set(coArray.map(JSON.stringify)), JSON.parse)
	},
// 调用函数并且将结果保存到对象里面
	setFormilyValue: () => {
		const selectOptions = this.convertArrayToOptions(getCustomerInfo.data , 0)
		const obj = {
			"name":"customerInfo",
		}
		obj.customerCode = selectOptions
		obj.customerInfo = getCustomerInfo.data
		return obj
	},
```

### 将对象设置为表单初始值

![70373560126](C:\Users\cheng.zhou\Documents\markdown\image\1703735601268.png)



### 配置响应器

![70373567469](C:\Users\cheng.zhou\Documents\markdown\image\1703735674696.png)

![70373574169](C:\Users\cheng.zhou\Documents\markdown\image\1703735741692.png)

```js
$effect(() => {
  const customerCode = $deps.customer_code
  const customerInfo = $values.customerInfo
  function getOtherFieldByCode(Code) {
    for (const customer of customerInfo) {
      if (customer.customer_code === Code) {
        return customer.customer_name
      }
    }
    return "" // 未找到时返回空字符串
  }
  $self.value = getOtherFieldByCode(customerCode)
}, [$deps.customer_code])
```



## formily自增表格一些处理

### 每行字段间的联动

填写数量和单价自动计算总价

![70373591247](C:\Users\cheng.zhou\Documents\markdown\image\1703735912475.png)

![70374018324](C:\Users\cheng.zhou\Documents\markdown\image\1703740183241.png)

```js
$effect(() => {
  const result = $deps.outbound_count * $deps.sale_price_taxin || 0
  $self.value = result.toFixed(2)
}, [$deps.outbound_count, $deps.sale_price_taxin])
```

### 表外字段与自增表联动

表外统计金额合计

![70374034639](C:\Users\cheng.zhou\Documents\markdown\image\1703740346390.png)

#### 给整个表一个字段标识

![70374041862](C:\Users\cheng.zhou\Documents\markdown\image\1703740418621.png)

#### 在给需要求和的字段增加响应器

![70374049930](C:\Users\cheng.zhou\Documents\markdown\image\1703740499301.png)

![70374057719](C:\Users\cheng.zhou\Documents\markdown\image\1703740577195.png)

```js
$effect(() => {
  $form.query("sale_delivery_item").take((target) => {
    const result = (target?.value || []).reduce((prev, cur) => {
      return Number(prev) + Number(cur.total)
    }, 0)
    console.log(result)
    $form.setValuesIn("sum", result)
  })
}, [$deps.outbound_count, $deps.sale_price_taxin])
```

### 表外字段与多个表联动

将出库明细表金额总和与退货明细表金额总和求和

![70374285870](C:\Users\cheng.zhou\Documents\markdown\image\1703742858709.png)

#### 配置第二张表的求和字段的响应器

![70374296338](C:\Users\cheng.zhou\Documents\markdown\image\1703742963387.png)

![70374312532](C:\Users\CHENG~1.ZHO\AppData\Local\Temp\1703743125320.png)

```js
$effect(() => {
  const a = $form.query("sale_delivery_item").take((target) => {
    const result1 = (target?.value || []).reduce((prev, cur) => {
      return Number(prev) + Number(cur.total_amount)
    }, 0)
    return result1
  })
  const b = $form.query("sale_return_item").take((target) => {
    const result2 = (target?.value || []).reduce((prev, cur) => {
      return Number(prev) + Number(cur.total)
    }, 0)
    return result2
  })
  const result = b + a
  $form.setValuesIn("sum_amount_taxex", result)
}, [$deps.reco_number, $deps.sale_price_taxin])
```

### 自动创建表格

#### 将数据作为对象的属性保存到表单初始值里面

![70425867784](C:\Users\cheng.zhou\Documents\markdown\image\1704258677849.png)

#### 在表单中创建和该属性同名的自增表格

![70425876465](C:\Users\cheng.zhou\Documents\markdown\image\1704258764659.png)

#### 让表格的字段标识与数据的字段名称一样

![70425883839](C:\Users\cheng.zhou\Documents\markdown\image\1704258838399.png)

![70425890603](C:\Users\cheng.zhou\Documents\markdown\image\1704258906038.png)

#### 最终效果

![70425895236](C:\Users\cheng.zhou\Documents\markdown\image\1704258952361.png)

## formily 表格中联级选择器(Cascader)数据传入

```js
setCascaderOption: () => {
		var addressData = province.json
		// 转换为联级选择器源数据的格式
		var transformedData = addressData.map(function (province) {
			return {
				label: province.name,
				value: province.name,
				children: province.city.map(function (city) {
					return {
						label: city.name,
						value: city.name,
						children: city.area.map(function (area) {
							return {
								label: area,
								value: area
							};
						})
					};
				})
			};
		});
		return transformedData
	},
```

## formily自增表格中的选择器选项跟随表外字段变化而变化

![70444006850](C:\Users\cheng.zhou\Documents\markdown\image\1704440068507.png)

![70444009941](C:\Users\cheng.zhou\Documents\markdown\image\1704440099418.png)

```js
$effect(() => {
  function getSelectOptions(originalData, warehouseName) {
    let transformedData = {}
    originalData.forEach((item) => {
      // 如果name还没有被添加到transformedData中，则添加它
      if (!transformedData[item.name]) {
        transformedData[item.name] = {
          name: item.name,
          bin_code: [],
        }
      }
      // 向对应的name下的bin_code数组添加一个新的对象
      transformedData[item.name].bin_code.push({
        label: item.bin_code,
        value: item.bin_code,
      })
    })
    return transformedData[warehouseName].bin_code
  }

  const originalData = $values.binCode
  const warehouseName = $deps.outbound_warehouse
  const selectOptions = getSelectOptions(originalData, warehouseName)
  $self.dataSource = selectOptions
}, [$deps.outbound_warehouse])
```

## 自增表格数据跟随表外字段变化

![70444306634](C:\Users\cheng.zhou\Documents\markdown\image\1704443066344.png)

![70444309485](C:\Users\cheng.zhou\Documents\markdown\image\1704443094855.png)

```js
//给整个表添加响应事件
$effect(() => {
  const myArr = $values.saleOrderItem
  const code = $deps.sale_order_code
  const table = myArr.filter((item) => item.sale_order_code === code)
  $self.value = table
}, [$deps.sale_order_code])
```

