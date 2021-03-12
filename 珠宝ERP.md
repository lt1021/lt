# 珠宝ERP

​	

### 原料调出和出库到其他库

1. 获取库位信息

```java
@GetMapping("/storage/query_client")
    @ApiOperation("客户仓过滤客户目录")
    @ApiImplicitParam(value = "仓库标识", name = "type")
    public ResponceData queryClient(@RequestParam Map<String, Object> map) {
        map.put("isClient",1);
        List<Storage> storages = storageService.query(map);
        StringBuffer sb = new StringBuffer();
        storages.stream().filter(d->d!=null).forEach(d->{
            if ("".equals(sb.toString()) || sb == null){
                sb.append(d.getClientId());
            }else {
                sb.append(","+d.getClientId());
            }
        });
        map.remove("type");
        map.put("supplierIds", sb.toString());
        map.put("waitDelete", 0);
        return clientService.queryPage(map).dataToJson();
    }
```

2. 达到



2. 3

   





```java
@GetMapping("crmClient")
@ApiOperation(value = "分页查询客户管理")
@ApiImplicitParam(value = "页码：pageNo，页大小：limit，" +
        " 分页条件：type（公司类型 1.参展客户 2.主力客户 3.商网客户 4.终端客户 5.贸易客户 6.转单客户）," +
        "waitDelete (作废状态，0代表正常，1代表作废)" +
        "搜索条件：nameCn、nameEn、nameCn)")
public ResponceData<BaseClient> queryPage(@RequestParam Map<String, Object> map) {
    if (map.get("isStorage") != null) {
        Map<String, Object> hashMap = new HashMap<>(3);
        hashMap.put("isClient", true);
        hashMap.put("type", map.get("storageType"));
        hashMap.put("delete", 0);
        map.put("clientIds", storageService.query(hashMap).stream().map(Storage::getClientId).distinct().map(String::valueOf).collect(Collectors.joining(",")));
    }
    //如果是订单查看客户,业务员只能看自己的
    if (map.get("isOrder") != null && !ThreadMapUtil.isAdminOrRoot()) {
        HrStaff staff = ContextUtil.getStaff(ThreadMapUtil.getStaffId());
        if (staff != null && staff.getRoleId() != null
                && (Arrays.stream(staff.getRoleId().split(",")).filter(d -> StringHelp.ifExists("1,2,3,4,10", d, ",")).count() > 0)) {
            if (staff.getRoleId().toString().equals("2")) {
                map.put("salerId", staff.getId());
            }
        } else {
            return ResponceData.fail(ExceptionMessage.getAuthNot());
        }
    }
    return service.queryPage(map).dataToJson();
}
```





## Excel表格

1. **前端传的数据**

2. **Map<String,Object>([0]{lang:cn},[1]{fileName:PD003},**
   		   **[2]{exportSort:2},[3]{currencyId:5},**
      		   **[4]{exportModelId:58}，[5]{billId:3},**
      		   **[6]{path:/storage/shipment/yf-ship2})**

3. **Excel需要的数据**

   Map<String,Object>(**
   **Object{[其他数据，list,clientStones,factoyStones],**
          **[其他数据，list,clientStones,factoyStones]})**

4. **其他数据：**
   	**客户：clientName**
      	**导出类型：exportType**
      	**日期：date**

5. **list：**
   	**货号：productName** 
      	**品名 productType**
      	**成色：alloy**
      	**手寸：sizeLenght**
      	**总重:prouctWeight**
      	**净金重：productNetWeight**
      	**金价：goldValue**
      	**损耗：wastage**
      	**基本工费：fees**
      	**金额：goldValues**
      	**日期：sendDate**
      	**单号：singleCode**
      	**出货图：image**
      	**数量：sendAmount**
   **clientStones：**
   	**石号：stoneCode**
   	**种类：stoneType**
   	**粒：stoneAmount**
   	**重量：stoneWeight**
   	**单价：stoneValue**
   **factoyStones**
   	**石号：stoneCode**
   	**种类：stoneType**
   	**粒：stoneAmount**
   	**重量：stoneWeight**
   	**金额：stonePrice**
   
6. ```java
   @Override
       public Object[] exportExcel(Map<String, Object> map) {
           List<Map<String, Object>> exportList = new ArrayList<>();
           StorageProductBill bill = mapper.get(Long.valueOf(map.get("billId").toString()));//单据
           List<StorageProductDetail> storageProductDetailList = detailService.query(map);//出货明细
           LinkedHashMap<String, Map<String, Object>> productMap = new LinkedHashMap<>();//
           BaseClient client = baseClientService.get(bill.getClientId());//客户信息
           List<Map<String, Object>> clientStoneList = new ArrayList<>();//客戶石头
           List<Map<String, Object>> factoyStoneList = new ArrayList<>();//工厂石头
           OrderExchangeRate orderExchangeRate = null;//订单汇率信息
           OrderIngredientPrice orderIngredientPrice = null;//主料价格信息
           String materialName = ""; //原料名称
           String clientCode = ""; //客户码
           String orderInfoCode = "";//订单码
   
           for (StorageProductDetail storageProductDetail : storageProductDetailList) {
               Map<String, Object> exportMap = new HashMap<>();
               OrderBatchProduct batchProduct = new OrderBatchProduct();
               OrderBatchProductSingle productSingle = null;
               if (storageProductDetail.getSingleId() != null) {
                   productSingle = singleService.get(storageProductDetail.getSingleId()); //分单号ID
                   batchProduct = orderBatchProductService.get(productSingle.getProductId());//批次产品id
               } else {
                   batchProduct.setBatchId(storageProductDetail.getBatchId()); //批次ID
                   batchProduct.setProductId(storageProductDetail.getProductId());//通过产品档案ID去查找产品编号，罗列在接收或发出页面上
                   batchProduct = orderBatchProductService.getBatchProduct(batchProduct);//批次产品信息
               }
               OrderProductInfo productInfo = orderProductInfoService.get(batchProduct.getProductId());//产品信息
               OrderInside inside = orderInsideService.getByProductId(productInfo.getId(), null);//产品内页信息
               ParamUnitDetail unitDetail = paramUnitDetailService.get(productInfo.getMeteringUnitId());//计量单位
               OrderInfo orderInfo = orderInfoService.get(batchProduct.getOrderId());//订单信息
               ParamAlloy alloy = paramAlloyService.get(productInfo.getAlloyId());//成色信息
               ParamMaterial material = paramMaterialService.get(alloy.getMaterialId());//材质信息
               if (materialName.equals("")) {
                   materialName = material != null ? material.getName() : "";
               }
               ParamProductType type = paramProductTypeService.get(productInfo.getProductCategory());//产品分类
               orderExchangeRate = new OrderExchangeRate();
               orderIngredientPrice = orderIngredientPriceService.getOrderIngredPrice(null, productInfo.getOrderId(), productInfo.getAlloyId());//主料信息,成色ID getAlloyId,订单id getOrderId
               orderExchangeRate.setRateType(1);//汇率类型(1=计价汇率，2=结算汇率)
               orderExchangeRate.setOrderId(productInfo.getOrderId());//订单id
               orderExchangeRate.setValuationId(Long.parseLong(map.get("currencyId").toString()));//货币
               orderExchangeRate = orderExchangeRateService.getBean(orderExchangeRate);//汇率信息
               map.put("orderId", productInfo.getOrderId());
               BigDecimal gemStoneValue = BigDecimal.ZERO; //宝石金额
               BigDecimal mainStoneValue = BigDecimal.ZERO;//主石金额
               BigDecimal deputyStoneValue = BigDecimal.ZERO;//副石金额
               BigDecimal mainStoneWeight = BigDecimal.ZERO;//主石重CT
               BigDecimal gemStoneWeight = BigDecimal.ZERO; //宝石重CT
               BigDecimal deputyStoneWeight = BigDecimal.ZERO;//副石重CT
               BigDecimal mainStoneAmount = BigDecimal.ZERO;/// 主石粒数
               BigDecimal deputyStoneAmount = BigDecimal.ZERO;//副石粒数
               BigDecimal deputyStoneFee = BigDecimal.ZERO;   ///// 副石镶工费
               BigDecimal mainStoneSettingValue = BigDecimal.ZERO;//主石镶工费
               BigDecimal deputyDiamondOneValue = BigDecimal.ZERO; //2.0 以上副钻石金额
               BigDecimal deputyDiamondTwoValue = BigDecimal.ZERO; //1.9-1.2 副钻石金额
               BigDecimal deputyDiamondThreeValue = BigDecimal.ZERO;//1.2以下副钻石金额
               BigDecimal deputyDiamondOneWeight = BigDecimal.ZERO; //2.0 以上副钻石重CT
               BigDecimal deputyDiamondTwoWeight = BigDecimal.ZERO; //1.9-1.2 副钻石重CT
               BigDecimal deputyDiamondThreeWeight = BigDecimal.ZERO;//1.2以下副钻石重CT
               List<Map<String, Object>> stones = new ArrayList<>();
               List<Map<String, Object>> allStones = new ArrayList<>();
               List<Map<String, Object>> deputyStones = new ArrayList<>();
               Integer mainCode = 0;
               Integer deputyCode = 0;
               if (productSingle != null) {
                   map.put("singleId", productSingle.getId());
               }
               map.put("productId", productSingle != null ? batchProduct.getId() : productInfo.getId());
               List<OrderBatchSingleStone> singleStoneList = productSingle == null ? null : singleStoneService.query(map);
               List<OrderProductStone> stoneList = productSingle != null ? null : orderProductStoneService.query(map);//该产品所有石头
               for (Object object : productSingle == null ? stoneList : singleStoneList) {
                   OrderProductStone productStone = new OrderProductStone();
                   OrderBatchSingleStone singleStone = new OrderBatchSingleStone();
                   BeanUtils.copyProperties(object, productSingle == null ? productStone : singleStone);
                   OrderBatchProductStone batchStone = new OrderBatchProductStone();
                   if (productSingle == null) {
                       batchStone.setProductId(batchProduct.getId());
                   } else {
                       batchStone.setSingleId(productSingle.getId());
                   }
                   batchStone.setStoneId(productSingle != null ? singleStone.getStoneId() : productStone.getStoneId());
                   batchStone = obpStoneService.getBean(batchStone);//批次石头
                   FileStone stone = fileStoneService.get(productSingle != null ? singleStone.getStoneId() : productStone.getStoneId());//石头档案
                   BaseStoneSpec stoneSpec = stone.getSpecId() == null ? null : baseStoneSpecService.get(stone.getSpecId()); //石头规格
                   BaseStoneType stoneType = stone.getTypeId() == null ? null : baseStoneTypeService.get(stone.getTypeId()); //石头类型
                   BaseStoneType stoneShape = stone.getShapeId() == null ? null : baseStoneShapeService.get(stone.getShapeId());//石头形状
                   Pattern pattern = Pattern.compile("-?[0-9]+\\.?[0-9]*");
                   Matcher matcher = stoneSpec == null ? null : pattern.matcher(stoneSpec.getName());
                   //石头价格
                   OrderStonePrice stonePrice = orderStonePriceService.getByOrderAndStoneId(productInfo.getOrderId(), productSingle != null ? singleStone.getStoneId() : productStone.getStoneId());
                   //石头镶工工费
                   OrderSettingFee settingFee = new OrderSettingFee();
                   settingFee.setOrderId(productInfo.getOrderId());
                   settingFee.setStoneId(productSingle != null ? singleStone.getStoneId() : productStone.getStoneId());
                   settingFee.setSettingId(productSingle != null ? singleStone.getSettingId() : productStone.getSettingId());
                   settingFee = orderSettingFeeService.get(settingFee);
                   if (batchStone != null) {
                       Map<String, Object> stoneMap = new HashMap<>();
                       stoneMap.put("stoneId", stone.getId());//石头id
                       stoneMap.put("stoneCode", stone.getCode());//石头编号
                       stoneMap.put("stoneName", stone.getName());//石头名称
                       stoneMap.put("sellValuation", stone.getSellValuation());//石头计价方式
                       stoneMap.put("stoneType", stoneType != null ? stoneType.getName() : "");//石头类型
                       stoneMap.put("stoneValue", stonePrice == null ? BigDecimal.ZERO : stonePrice.getPrice());//石头价格
                       stoneMap.put("stoneAmount", productSingle != null ? singleStone.getAmount() : productStone.getAmount());//石头数量
                       stoneMap.put("stonePrice", stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));//石头价格
                       stoneMap.put("settingFee", settingFee == null ? BigDecimal.ZERO : settingFee.getPrice());//镶工工费
                       stoneMap.put("stoneWeight", (batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount())).multiply(BigDecimal.valueOf(5)).setScale(3, BigDecimal.ROUND_DOWN));//石头重
                       if (stone.getSupplyChannel().equals(3)) {
                           factoyStoneList.add(stoneMap);//自供石
                       } else {
                           clientStoneList.add(stoneMap);//客供石
                       }
                       if (productSingle != null ? singleStone.getPrioritize().equals(1) : productStone.getPrioritize().equals(1)) {
                           allStones.add(stoneMap);//主石头
                       } else {
                           deputyStones.add(stoneMap);//次石头
                       }
                   }
                   if (batchStone != null && (productSingle != null ? singleStone.getPrioritize().equals(1) : productStone.getPrioritize().equals(1))) {
                       mainStoneAmount = mainStoneAmount.add(productSingle != null ? singleStone.getAmount() : productStone.getAmount());
                       mainStoneValue = mainStoneValue.add(stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));
                       mainStoneSettingValue = mainStoneSettingValue.add(settingFee == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(settingFee.getPrice()));
                       exportMap.put("mainStonePrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("mainStonePrice") ? exportMap.get("mainStonePrice") : stonePrice == null ? BigDecimal.ZERO : stonePrice.getPrice());
                       mainStoneWeight = mainStoneWeight.add(batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                       exportMap.put("mainStoneSettingPrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("mainStoneSettingPrice") ? exportMap.get("mainStoneSettingPrice") : stonePrice == null ? BigDecimal.ZERO : settingFee.getPrice());
                       mainCode++;
                       exportMap.put("mainStoneCode" + mainCode, stone.getCode());//主编号
                       exportMap.put("mainStoneType" + mainCode, stoneType != null ? stoneType.getName() : ""); /// 主石头类型
                       exportMap.put("mainStoneShape" + mainCode, stoneShape != null ? stoneShape.getName() : "");//主石头形状
                       exportMap.put("mainStoneAmount" + mainCode, (productSingle != null ? singleStone.getAmount() : productStone.getAmount()));//主石头数量
                       exportMap.put("mainStonePrice" + mainCode, stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));//石头价格
                       exportMap.put("mainStoneWeight" + mainCode, (batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount())).multiply(BigDecimal.valueOf(5)).setScale(3, BigDecimal.ROUND_DOWN));//主石头重量
                   } else if (stoneType != null && stoneSpec != null && batchStone != null && (productSingle != null ? singleStone.getPrioritize().equals(0) : productStone.getPrioritize().equals(0)) && stoneType.getNameCn().equals("钻石") && matcher.matches() == true) {
                       if (Double.parseDouble(stoneSpec.getName()) >= 2.0) {
                           deputyDiamondOneValue = deputyDiamondOneValue.add(stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));
                           exportMap.put("diamondOnePrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("diamondOnePrice") ? exportMap.get("diamondOnePrice") : stonePrice == null ? BigDecimal.ZERO : stonePrice.getPrice());
                           deputyDiamondOneWeight = deputyDiamondOneWeight.add(batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                       } else if (Double.parseDouble(stoneSpec.getName()) >= 1.2 && Double.parseDouble(stoneSpec.getName()) <= 1.9) {
                           deputyDiamondTwoValue = deputyDiamondTwoValue.add(stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));
                           exportMap.put("diamondTwoPrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("diamondTwoPrice") ? exportMap.get("diamondTwoPrice") : stonePrice == null ? BigDecimal.ZERO : stonePrice.getPrice());
                           deputyDiamondTwoWeight = deputyDiamondTwoWeight.add(batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                       } else if (Double.parseDouble(stoneSpec.getName()) <= 1.2) {
                           deputyDiamondThreeValue = deputyDiamondThreeValue.add(stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));
                           exportMap.put("diamondThreePrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("diamondThreePrice") ? exportMap.get("diamondThreePrice") : stonePrice == null ? BigDecimal.ZERO : stonePrice.getPrice());
                           deputyDiamondThreeWeight = deputyDiamondThreeWeight.add(batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                       }
                       deputyStoneValue = deputyStoneValue.add(stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));
                       deputyStoneWeight = deputyStoneWeight.add(batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                   } else if (stoneType != null && batchStone != null && stoneType.getNameCn().equals("宝石")) {
                       gemStoneValue = gemStoneValue.add(stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));
                       exportMap.put("gemStonePrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("gemStonePrice") ? exportMap.get("gemStonePrice") : stonePrice == null ? BigDecimal.ZERO : stonePrice.getPrice());
                       gemStoneWeight = gemStoneWeight.add(batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                   }
                   if (batchStone != null && (productSingle != null ? singleStone.getPrioritize().equals(0) : productStone.getPrioritize().equals(0))) {
                       deputyCode++;
                       deputyStoneAmount = deputyStoneAmount.add((productSingle != null ? singleStone.getAmount() : productStone.getAmount()));
                       deputyStoneFee = deputyStoneFee.add(settingFee == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(settingFee.getPrice()));
                       exportMap.put("deputyStoneCode" + deputyCode, stone.getCode());//辅石头编号
                       exportMap.put("deputyStoneType" + deputyCode, stoneType != null ? stoneType.getName() : ""); /// 辅石头类型
                       exportMap.put("deputyStoneShape" + deputyCode, stoneShape != null ? stoneShape.getName() : "");//辅石头形状
                       exportMap.put("deputyStoneAmount" + deputyCode, (productSingle != null ? singleStone.getAmount() : productStone.getAmount()));//辅石头数量
                       exportMap.put("deputyStonePrice" + deputyCode, stonePrice == null ? BigDecimal.ZERO : (productSingle != null ? singleStone.getAmount() : productStone.getAmount()).multiply(stonePrice.getPrice()));//辅石头单价
                       exportMap.put("deputyStoneWeight" + deputyCode, (batchStone.getSendoutAmount().compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : batchStone.getSendoutWeight().divide(batchStone.getSendoutAmount(), 10, BigDecimal.ROUND_HALF_UP).multiply(productSingle != null ? singleStone.getAmount() : productStone.getAmount())).multiply(BigDecimal.valueOf(5)).setScale(3, BigDecimal.ROUND_DOWN));//辅石头重量
                   }
               }
               Integer size = productMap.size() == 0 ? 2 : productMap.size() + 2;
               for (int i = 1; i <= 3; i++) {
                   if (exportMap.containsKey("mainStoneCode" + i)) {
                       Map<String, Object> stoneMap = new HashMap<>();
                       stoneMap.put("stoneType", exportMap.get("mainStoneType" + i));//石头类型
                       stoneMap.put("stoneWeight", exportMap.get("mainStoneWeight" + i));//石头重
                       stoneMap.put("stoneAmount", exportMap.get("mainStoneAmount" + i));//石头数量
                       stoneMap.put("stoneCode", i == 1 ? "出货明细!T" + size + "" : i == 2 ? "出货明细!X" + size + "" : "出货明细!AB" + size + "");//石头编号
                       stones.add(stoneMap);
                   }
               }
               for (int i = 1; i <= deputyCode; i++) {
                   if (exportMap.containsKey("deputyStoneCode" + i)) {
                       Map<String, Object> stoneMap = new HashMap<>();
                       stoneMap.put("stoneType", exportMap.get("deputyStoneType" + i));//石头类型
                       stoneMap.put("stoneWeight", exportMap.get("deputyStoneWeight" + i));//石头重
                       stoneMap.put("stoneAmount", exportMap.get("deputyStoneAmount" + i));//石头数量
                       stoneMap.put("stoneCode", i == 1 ? "AH" : i == 2 ? "AN" : i == 3 ? "AT" : i == 4 ? "AZ"
                               : i == 5 ? "BF" : i == 6 ? "BL" : i == 7 ? "BR" : i == 8 ? "BX" : i == 9 ? "CD" : "CJ");//石头编号
                       stoneMap.put("stoneCode", stoneMap.get("stoneCode").toString() + size);
                       stones.add(stoneMap);
                   }
               }
               //执版的查询
               ParamSysProcess processes = new ParamSysProcess();
               processes.setNameCn("执版");
               Integer sysProcessId = paramSysProcessService.check(processes);
               List<ParamProcess> processList = paramProcessService.getIdBySysProcessId(sysProcessId);
               Long processId = Long.valueOf(0);
               if (processList.size() > 0) {//执版/工费
                   for (ParamProcess process : processList) {
                       processId = ((processId == 0) ? process.getId() : processId >= process.getId() ? process.getId() : processId);
                   }
               }
               //配件信息
               map.put("productId", productInfo.getId());
               List<Map<String, Object>> fittings = new ArrayList<>();
               //订单产品的配件信息
               List<OrderProductFitting> fittingList = orderProductFittingService.query(map);
               BigDecimal fittingValue = BigDecimal.ZERO;//18K配件金额
               for (OrderProductFitting fitting : fittingList) {
                   Map<String, Object> fittingMap = new HashMap<>();
                   FileFitting fileFitting = fileFittingService.get(fitting.getFittingId());
                   ParamAlloy fittingAlloy = fileFitting.getAlloyId() != null ? paramAlloyService.get(fileFitting.getAlloyId()) : null;
                   OrderFittingPrice fittingPrice = new OrderFittingPrice();
                   fittingPrice.setOrderId(productInfo.getOrderId());
                   fittingPrice.setFittingId(fitting.getFittingId());
                   fittingPrice = orderFittingPriceService.getByFitting(fittingPrice);
                   fittingValue = fittingValue.add(fittingPrice == null ? BigDecimal.ZERO : fitting.getAmount().multiply(fittingPrice.getFee()));
                   exportMap.put("fittingPrice", bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? !productMap.containsKey(productSingle.getId()) : !productMap.containsKey(batchProduct.getId() + "-" + productInfo.getId())) && exportMap.containsKey("fittingPrice") ? exportMap.get("fittingPrice") : fittingPrice == null ? BigDecimal.ZERO : fittingPrice.getFee());
                   fittingMap.put("fittingCode", fileFitting.getCode());//配件编号
                   fittingMap.put("fittingRadio", fittingAlloy != null ? fittingAlloy.getFittingRatio() : null);//配件折足
                   fittingMap.put("fittingWeight", fitting.getSingle().multiply(storageProductDetail.getProductAmount()));//配件重
                   //针对宝莱
                   Map<String, Object> bmap = new HashMap<>();
                   bmap.put("productId", batchProduct.getId());
                   bmap.put("fittingId", fileFitting.getId());
                   List<OrderBatchProductFitting> fs = orderBatchProductFittingService.query(bmap);
                   BigDecimal blWeight = fs.stream().filter(d -> d.getAlreadyAmount().compareTo(BigDecimal.ZERO) > 0).map(d -> d.getAlreadyWeight().divide(d.getAlreadyAmount(), 6, BigDecimal.ROUND_HALF_UP).multiply(d.getAmount()).multiply(storageProductDetail.getProductAmount())).reduce(BigDecimal.ZERO, BigDecimal::add);
                   fittingMap.put("blFittingWeight", blWeight);//配件重
                   fittings.add(fittingMap);
               }
               //工序工费
               List<OrderStepFee> feeList = orderStepFeeService.query(map);
               BigDecimal version = BigDecimal.ZERO;// 起版
               BigDecimal fee = BigDecimal.ZERO;  //// 工费
               BigDecimal fees = BigDecimal.ZERO;//基本工费
               for (OrderStepFee stepFee : feeList) {
                   if (stepFee.getStepId().equals(processId)) {
                       version = version.add(stepFee.getPrice());
                   } else {
                       fee = fee.add(stepFee.getPrice());
                   }
                   fees = fees.add(stepFee.getPrice());
               }
               allStones.addAll(deputyStones);
               clientCode = orderInfo.getClientCodes();
               orderInfoCode = orderInfo.getCode();
               BaseClient clientInfo = baseClientService.get(orderInfo.getClientId());//客户信息
               ClientOrderConfig orderConfig = clientOrderConfigService.get(orderInfo.getClientId());//客户栏目配置
               CostCrmPrice costCrmPrice = costCrmPriceService.getProductId(productInfo.getProductId(), orderInfo.getClientId());
               BasePlating plating = productInfo.getPlatingId() == null ? null : basePlatingService.get(productInfo.getPlatingId());//电镀方式
               exportMap.put("fee", fee);// 工费（元）
               exportMap.put("fees", fees);//基本工费
               exportMap.put("version", version); // 起版
               exportMap.put("alloy", alloy.getName());//成色
               exportMap.put("fittings", fittings);//全部配件
               exportMap.put("stones", stones); // 部分石头集合
               exportMap.put("allStones", allStones);//全部石头
               exportMap.put("productId", productInfo.getId());//产品Id
               exportMap.put("memo", inside.getProduceCruces()); // 生产要点
               exportMap.put("produceCruces", storageProductDetail.getMemo()); // 明细备注
               exportMap.put("image", (StringUtils.isNotBlank(productInfo.getThumb1()) ? productInfo.getThumb1() : StringUtils.isNotBlank(productInfo.getThumb2()) ? productInfo.getThumb2() : productInfo.getThumb3()));//图片
               exportMap.put("gemStoneValue", gemStoneValue);// 宝石金额
               exportMap.put("fittingValue", fittingValue);//18K配件金额
               exportMap.put("infoCode", orderInfo.getCode());//订单编号
               exportMap.put("infoName", orderInfo.getName());//订单名称
               exportMap.put("mainStoneValue", mainStoneValue); // 主石金额
               exportMap.put("sendCode", bill.getSendBillCode());//发出单号
               exportMap.put("deputyStoneFee", deputyStoneFee); // 副石镶工费
               exportMap.put("mainStoneAmount", mainStoneAmount); // 主石粒数
               exportMap.put("deputyStoneValue", deputyStoneValue);//副石金额
               exportMap.put("productCode", productInfo.getCode());//产品编号
               exportMap.put("productInfoName", productInfo.getName());//产品名称
               exportMap.put("clientName", clientInfo.getClientShort());// 客户名
               exportMap.put("deputyStoneAmount", deputyStoneAmount); // 副石粒数
               exportMap.put("clientCode", orderInfo.getClientCodes());//客户单号
               exportMap.put("clientAddressTable", "CS" + size); // 客户地址（地区）
               exportMap.put("clientAddress", client.getAddress());//客户地址（地区）
               exportMap.put("productName", productInfo.getOriginalCode());//产品原始编号
               exportMap.put("mainStoneSettingValue", mainStoneSettingValue);//主石镶工费
               exportMap.put("productType", type != null ? type.getName() : "");//产品分类
               exportMap.put("grossWeight", storageProductDetail.getGrossWeight());// 连袋毛重
               exportMap.put("sendAmount", storageProductDetail.getProductAmount());//发出数量
               exportMap.put("plating", plating == null ? null : plating.getName());//电镀方式
               exportMap.put("stoneWeight", storageProductDetail.getStoneWeight());// 发出石重
               exportMap.put("jyxStoneWeight", storageProductDetail.getStoneWeight().multiply(BigDecimal.valueOf(5)));// 发出石重
               exportMap.put("deputyDiamondOneValue", deputyDiamondOneValue);//2.0 以上副钻石金额
               exportMap.put("deputyDiamondTwoValue", deputyDiamondTwoValue);//1.9-1.2 副钻石金额
               exportMap.put("productWeight", storageProductDetail.getProductWeight());//发出重量
               exportMap.put("fittingWeight", storageProductDetail.getFittingWeight());// 发出配件重
               exportMap.put("deputyDiamondThreeValue", deputyDiamondThreeValue);//1.2以下副钻石金额
               exportMap.put("singleCode", productSingle != null ? productSingle.getCode() : "");//分单号
               exportMap.put("quoteFinalFee", storageProductDetail.getQuoteFinalFee());//单件工费（工费单价）
               exportMap.put("amountUnit", unitDetail != null ? unitDetail.getNameShort() : null);//计量单位
               exportMap.put("sizeLenght", ContextUtil.parseSize(storageProductDetail.getSizeLengthSet()));// 手寸
               exportMap.put("clientProductCode", costCrmPrice != null ? costCrmPrice.getClientCodes() : null);//客户款号
               exportMap.put("wastage", orderIngredientPrice != null ? orderIngredientPrice.getWastage() : BigDecimal.ZERO);//损耗，耗率
               exportMap.put("sendDate", orderInfo.getShipmentDate() == null ? "" : DateHelp.formats(orderInfo.getShipmentDate()));//发出日期
               exportMap.put("productNetWeight", storageProductDetail.getProductWeight().subtract(storageProductDetail.getStoneWeight()));//净金重
               exportMap.put("gemStoneWeight", (gemStoneWeight.multiply(BigDecimal.valueOf(5))).setScale(3, BigDecimal.ROUND_DOWN)); // 宝石重CT
               exportMap.put("mainStoneWeight", (mainStoneWeight.multiply(BigDecimal.valueOf(5))).setScale(3, BigDecimal.ROUND_DOWN));//主石重CT
               exportMap.put("deputyStoneWeight", (deputyStoneWeight.multiply(BigDecimal.valueOf(5))).setScale(3, BigDecimal.ROUND_DOWN));//副石重CT
               exportMap.put("productRatio", storageProductDetail.getProductWeight().subtract(storageProductDetail.getStoneWeight()).multiply(alloy.getProductRatio()));//折足重-yf
               exportMap.put("alloyProductRatio", alloy.getProductRatio());//成分
               exportMap.put("deputyDiamondOneWeight", (deputyDiamondOneWeight.multiply(BigDecimal.valueOf(5))).setScale(3, BigDecimal.ROUND_DOWN));//2.0 以上副钻石重CT
               exportMap.put("deputyDiamondTwoWeight", (deputyDiamondTwoWeight.multiply(BigDecimal.valueOf(5))).setScale(3, BigDecimal.ROUND_DOWN));//1.9-1.2 副钻石重CT
               exportMap.put("deputyDiamondThreeWeight", (deputyDiamondThreeWeight.multiply(BigDecimal.valueOf(5))).setScale(3, BigDecimal.ROUND_DOWN));//1.2以下副钻石重CT
               exportMap.put("allCode", ("分单:" + batchProduct.getId()) + ("\n款号:" + productInfo.getCode()) + ("\n货号:" + bill.getSendBillCode()));//所有单号
               exportMap.put("stoneFinalPriceTotal", storageProductDetail.getStoneFinalPriceTotal());//石值/件
               exportMap.put("processFinalFeeTotal", storageProductDetail.getProcessFinalFeeTotal());//杯底/件
               exportMap.put("settingFinalFeeTotal", storageProductDetail.getSettingFinalFeeTotal());//镶工/件
               exportMap.put("clientWastage", orderConfig != null ? orderConfig.getProcessRate().divide(BigDecimal.valueOf(100), 6, BigDecimal.ROUND_HALF_UP) : BigDecimal.ZERO); // 客户加工耗率
               BigDecimal goldValue = (bill.getIngredientPrice().compareTo(BigDecimal.ZERO) > 0 ? bill.getIngredientPrice() : (orderIngredientPrice != null ? orderIngredientPrice.getPrice() : BigDecimal.ZERO).multiply(alloy.getProductRatio()));//金价
               exportMap.put("ingredientPrice", bill.getIngredientPrice().compareTo(BigDecimal.ZERO) > 0 ? bill.getIngredientPrice() : (orderIngredientPrice != null ? orderIngredientPrice.getPrice() : BigDecimal.ZERO).multiply(orderIngredientPrice != null ? orderIngredientPrice.getWastage().add(BigDecimal.ONE) : BigDecimal.ONE));//金价（连耗）
               exportMap.put("goldValue", (!(alloy.getName().equals("18K黄") || alloy.getName().equals("18K红")) ? goldValue : goldValue.add(BigDecimal.valueOf(0.7))).setScale(2, BigDecimal.ROUND_DOWN));//金价（当成色为18K黄或18K红时：金价=（金价+(折足净重*0.7)））
               exportMap.put("embryoPrice", storageProductDetail.getBasisFinalPriceTotal().add(storageProductDetail.getSettingFinalFeeTotal()).add(storageProductDetail.getFittingFinalPriceTotal()).add(storageProductDetail.getFittingFinalFeeTotal()).add(storageProductDetail.getProcessFinalFeeTotal()));//胚底单价
               exportMap.put("goldValues", ((storageProductDetail.getProductWeight().subtract(storageProductDetail.getStoneWeight())).multiply(!(alloy.getName().equals("18K黄") || alloy.getName().equals("18K红")) ? goldValue : goldValue.add(BigDecimal.valueOf(0.7))).add(fees)).setScale(2, BigDecimal.ROUND_DOWN));
               productMap.put(bill.getBillType().equals(2) ? ("PD" + storageProductDetail.getCode()) : (productSingle != null ? productSingle.getId().toString() : batchProduct.getId() + "-" + productInfo.getId().toString()), exportMap);
           }
           for (String key : productMap.keySet()) {
               exportList.add(productMap.get(key));
           }
           this.productSort(map, exportList);//排序
           ParamUnitDetail paramUnitDetail = paramUnitDetailService.get(orderExchangeRate != null ? orderExchangeRate.getSettlementId() : 0);
           Map<String, Object> exportMap = new HashMap<>();
           exportMap.put("list", exportList);//数据
           exportMap.put("clientCode", clientCode);//客户单号
           exportMap.put("orderInfoCode", orderInfoCode);//订单号
           exportMap.put("sendMemo", bill.getMemo());// 单据备注
           exportMap.put("materialName", materialName);//产品材质
           exportMap.put("clientStones", clientStoneList);//客供石
           exportMap.put("factoyStones", factoyStoneList);//厂供石
           exportMap.put("sendCode", bill.getSendBillCode()); //发出单号
           exportMap.put("date", DateHelp.formats(new Date())); //// 日期
           exportMap.put("clientName", client.getClientShort());// 客户名
           exportMap.put("companyName", client.getName()); //客户公司名称
           exportMap.put("clientAddress", client.getAddress());//客户公司地址
           exportMap.put("sendDate", DateHelp.formats(bill.getSendDate())); //日期
           exportMap.put("cDate", DateHelp.formats(bill.getCdate())); //创建日期
           exportMap.put("sendName", ContextUtil.getPersonName(bill.getSendUserId())); //发出人
           exportMap.put("currency", paramUnitDetail != null ? paramUnitDetail.getName() : "人民币");//币种
           exportMap.put("exportType", bill.getBillType().equals(0) ? "按产品" : bill.getBillType().equals(1) ? "按分单" : "按每件");
           exportMap.put("wastage", orderIngredientPrice != null ? orderIngredientPrice.getWastage().multiply(BigDecimal.valueOf(100)) : BigDecimal.ZERO);//损耗，耗率
           exportMap.put("goldPrice", bill.getIngredientPrice().compareTo(BigDecimal.ZERO) > 0 ? bill.getIngredientPrice() :
                   orderIngredientPrice != null ? orderIngredientPrice.getPrice().multiply(orderIngredientPrice.getWastage().add(BigDecimal.ONE)) : BigDecimal.ZERO);//主料单价（连耗）
           return new Object[]{exportMap};
   
       }
   
       @Override
       public Object[] exportSWDZExcel(Map<String, Object> map) throws IOException {
   //        Object[] object = this.exportExcel(map);
   //        Map<String, Object> storageMap = (Map<String, Object>) object[0];
   //        return new Object[]{storageMap};
           List<Map<String, Object>> exportList = new ArrayList<>();
           StorageProductBill bill = mapper.get(Long.valueOf(map.get("billId").toString()));//单据
           List<StorageProductDetail> storageProductDetailList = detailService.query(map);//出货明细
           LinkedHashMap<String, Map<String, Object>> productMap = new LinkedHashMap<>();//
           for (StorageProductDetail detail:storageProductDetailList){
               Map<String, Object> exportMap = new HashMap<>();
           }
   
   
           
           return new Object[]{};
       }
   ```

   

### 导出 斯维迪智

客户名称
单号
交单日期
客户公司
客户地址

list
		款号，图片，单价 数量 价格 备注 ，

| 模板路径 | storage/shipment/productStorage3DImg |
| -------- | ------------------------------------ |
| 访问url  | storage/product/swdz/exportExcel     |

```java
@PostMapping("storage/product/swdz/exportExcel")
    @ApiOperation(value = "导出成品仓出货发票")
    @Export
    public ResponceData exportSWDZExcel(@RequestParam Map<String, Object> map) {
        Method method = null;
        try {
            method = service.getClass().getMethod("exportSWDZExcel", Map.class);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
            return ResponceData.fail(ExceptionMessage.getSelectNot());
        }
        if (map.get("path") == null || method == null) {
            return ResponceData.fail(ExceptionMessage.getSelectNot());
        }
        String lang = "_" + map.get("lang").toString();
        return ResponceData.data(DowloadExcel.exportExcel(map.get("path").toString(), lang, service, method, true, map));
    }
```

```java
 @Override
    public Object[] exportSWDZExcel(Map<String, Object> map) throws IOException {
        List<Map<String, Object>> exportList = new ArrayList<>();
        StorageProductBill bill = mapper.get(Long.valueOf(map.get("billId").toString()));//单据
        List<StorageProductDetail> storageProductDetailList = detailService.query(map);//出货明细
        LinkedHashMap<String, Map<String, Object>> productMap = new LinkedHashMap<>();//
        BaseClient client = baseClientService.get(bill.getClientId());//客户信息
        OrderInfo orderInfo = null;//订单信息
        for (StorageProductDetail storageProductDetail : storageProductDetailList) {
            Map<String, Object> exportMap = new HashMap<>();
            OrderBatchProduct batchProduct = new OrderBatchProduct();
            OrderBatchProductSingle productSingle = null;
            if (storageProductDetail.getSingleId() != null) {
                productSingle = singleService.get(storageProductDetail.getSingleId()); //分单号ID
                batchProduct = orderBatchProductService.get(productSingle.getProductId());//批次产品id
            } else {
                batchProduct.setBatchId(storageProductDetail.getBatchId()); //批次ID
                batchProduct.setProductId(storageProductDetail.getProductId());//通过产品档案ID去查找产品编号，罗列在接收或发出页面上
                batchProduct = orderBatchProductService.getBatchProduct(batchProduct);//批次产品信息
            }
            OrderProductInfo productInfo = orderProductInfoService.get(batchProduct.getProductId());//产品信息
            orderInfo = orderInfoService.get(batchProduct.getOrderId());//订单信息
            exportMap.put("productCode", productInfo.getCode());//款号
            exportMap.put("image", productInfo.getThumb1());//图片
            exportMap.put("quoteFinalFee", storageProductDetail.getQuotePrice());//单件单价
            exportMap.put("sendAmount", storageProductDetail.getProductAmount());//数量
            exportMap.put("produceCruces", storageProductDetail.getMemo());//备注
            ExcelHelp.findLogo(exportMap, productInfo.getThumb1());
            exportList.add(exportMap);
        }
        this.productSort(map, exportList);//排序
        Map<String, Object> exportMap = new HashMap<>();
        exportMap.put("list", exportList);//数据
        exportMap.put("sendCode", bill.getSendBillCode());//单号
        exportMap.put("clientCode", orderInfo == null ? null : orderInfo.getClientCodes());//单号
        exportMap.put("clientName", client.getClientShort()); //客户
        exportMap.put("sendDate", DateHelp.formats(bill.getSendDate())); //日期
        exportMap.put("companyName", client.getName()); //客户公司
        exportMap.put("clientAddress", client.getAddress()); //公司地址
        return new Object[]{exportMap};
    }
```

模板格式

$<<exportLists(1,12,1,10,1,14,14)

​	$<<list(8,8,1,10,1)

​	$>>

$>>







### Excel请求路径

#### 1. Quotation-AR询价单

​	访问路径：export/orderExport

​	模板路径：inquiry/Quotation-AR-inquiry

#### 2. 出货装箱单

​	访问路径：PdStorage/findProduceStorageBillExport

​	模板路径：/storage/shipment/productStorage

#### 3. 银姿出货清单

​	访问路径：PdStorage/findProduceStorageBillExportYZ

​	模板路径：storage/shipment/productStorageYZ

#### 4. 鑫亿澳宝出货清单

​	访问路径：PdStorage/findProduceStorageBillExport

​	模板路径：storage/shipment/productStorageXYAB

#### 5. abs出货清单

​	访问路径：storage/shipment/productStorageABS

​	模板路径：storage/shipment/productStorageABS

#### 6. 壹梵出货单1

​	访问路径：/storage/product/exportExcel

​	模板路径：/storage/shipment/yf-ship1

#### 7. 壹梵出货单2

​	访问路径：/storage/product/exportExcel2

​	模板路径：/storage/shipment/yf-ship2

#### 8. 出货标签纸

​	访问路径：/storage/product/exportExcel

​	模板路径：/storage/shipment/yf-label

#### 9. 出货标签纸

​	访问路径：/storage/product/exportExcel

​	模板路径：/storage/shipment/productStorageXYAB

#### 10. 锋利出货单

​	访问路径：/storage/product/exportExcel

​	模板路径：/storage/shipment/productStorageFL

####   11. 生产订单

​	模板路径：produce/production_order_bl

​	访问路径：order/workOrderPrint/exportExcel

####  12. 配石单-按产品

​	模板路径：produce/productStone

​	访问路径：order/productStone/exportExcel

#### 13. 配件单-按产品

​	模板路径：produce/productFitting

​	访问路径：order/productStone/exportExcel

#### 14.配石单-按分单

​	模板路径：produce/splitstone

​	访问路径：order/singleStone/exportExcel

#### 15. 配件单-按分单

​	模板路径：produce/splitfitting

​	访问路径：order/singleStone/exportExcel

#### 16. 胶模单-按订单

​	模板路径：produce/plan/ordermould

​	访问路径：clientOrder/exportMouldByOrder

#### 17. 订单编号

​	模板路径：produce/plan/produce_codes

​	访问路径：/orderQuoteController/exportOrderAllCodes

#### 18. HT生产单

​	模板路径：/produce/plan/orderProductHT

​	访问路径：orderQuoteController/exportOrderProductHT

#### 19. 订单产品图片

​	模板路径：produce/plan/produce-order-pcs

​	访问路径：orderQuoteController/exportOrderProductThumb

#### 20. Quotation-AR确认单

​	模板路径：confirm/Quotation-AR-order

​	访问路径：/order/clientOrder/exportExcel

### 成品出货的 根据收货记录 和订单批次批量添加明细

前端传的参数

DY货品移交出货Excel导出

```java
@PostMapping("/storage/product/handover/dy/shipment/exportExcel")
@Export
public ResponceData exportExcelDyShipmentProduct(@RequestParam Map<String, Object> map) {
    Method method = null;
    try {
        method = service.getClass().getMethod("queryStorageDyShipmentExport", Map.class);
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
        return ResponceData.fail(ExceptionMessage.getSelectNot());
    }
    if (map.get("path") == null || method == null) {
        return ResponceData.fail(ExceptionMessage.getSelectNot());
    }
    String lang = "_" + map.get("lang").toString();
    return ResponceData.data(DowloadExcel.exportExcel(map.get("path").toString(), lang, service, method, true, map));
}
```

```java
@Override
public Object[] queryStorageDyShipmentExport(Map<String, Object> map) throws ParseException{
    map.put("timeSort", 0);
    List<AnalyzeProductHandover> data = (List<AnalyzeProductHandover>) this.queryStorageProductHandover(map).getData();
    BigDecimal sum = BigDecimal.ZERO;//合计金额大写
     for (AnalyzeProductHandover analyze : data) {
         BigDecimal deputyStoneAmount = BigDecimal.ZERO;//副石粒数
         BigDecimal deputyStoneValue = BigDecimal.ZERO;//副石价格
         Map<String,Object> opsmap = new HashMap<>();
         opsmap.put("productId",analyze.getProductId());
         opsmap.put("prioritize",0);
         List<OrderProductStone> stoneList =  orderProductStoneService.query(opsmap);//该产品所有石头
         for (OrderProductStone ops:stoneList) {
             deputyStoneAmount = deputyStoneAmount.add(ops.getAmount());
             deputyStoneValue = deputyStoneValue.add(ops.getPrice());
         }
        analyze.setDeputyStoneAmount(deputyStoneAmount);//副石粒
        analyze.setDeputyStoneValue(deputyStoneValue);//副石头价格
        analyze.setSendDateStr(analyze.getSendDate() == null ? null : DateHelp.formats(analyze.getSendDate()));
        sum = sum.add((analyze.getSilverPrice().multiply(analyze.getProductWeight()).add(deputyStoneAmount.multiply(deputyStoneValue)).add(analyze.getQuoteFinalFee()))).multiply(analyze.getProductAmount()) ;//总金额
    }
    map.put("list", BeanHelp.objectToMap(data));
    map.put("date",DateHelp.formats(data.get(0).getSendDate()));
    map.put("clientName", map.get("clientId") != null ? clientService.get(Long.valueOf(map.get("clientId").toString())).getName() : "");//客户名称
    map.put("sendName", data.get(0).getSendName());//发货单位
    map.put("sendUserName", data.get(0).getSendUserName());//发货人
    map.put("incomeUserId", ContextUtil.getPersonName( data.get(0).getIncomeUserId()));//收货人
    map.put("incomeId", ContextUtil.getDeptName(data.get(0).getIncomeId()));//收货人单位
    String sum1 = sum.toString();
    Double integer=Double.parseDouble(sum1);
    Integer s= integer.intValue();
    map.put("sum",convert(s));//大写总金额
    //汇总数据
    Map<String, Object> tmap = new HashMap<>();
    if (map.get("path").toString().contains("3DShipmentProductDetail")) {
        //分组算汇总
        Map<String, List<AnalyzeProductHandover>> listMap = data.stream().collect(Collectors.groupingBy(d -> d.getClientId() + "-" + d.getSendBillCode()));
        List<Map<String, Object>> totalList = listMap.values().stream().map(d -> {
            Map<String, Object> m = new HashMap<>();
            m.put("sendDate", d.get(0).getSendDate());
            m.put("sendDateStr", d.get(0).getSendDateStr());
            m.put("sendBillCode", d.get(0).getSendBillCode());
            m.put("clientName", d.get(0).getClientName());
            m.put("productAmount", d.stream().map(AnalyzeProductHandover::getProductAmount).reduce(BigDecimal.ZERO, BigDecimal::add));
            m.put("productWeight", d.stream().map(AnalyzeProductHandover::getProductWeight).reduce(BigDecimal.ZERO, BigDecimal::add));
            m.put("ingredientPrice", d.get(0).getIngredientPrice());
            return m;
        }).collect(Collectors.toList());
        totalList.sort((s1, s2) -> ((Date) s1.get("sendDate")).compareTo((Date) s2.get("sendDate")));
        tmap.put("clientName", map.get("clientId") != null ? clientService.get(Long.valueOf(map.get("clientId").toString())).getName() : "");
        tmap.put("list", totalList);
        tmap.put("date", map.get("date"));
        return new Object[]{map, tmap};
    } else {
        return new Object[]{map};
    }
}

private static final char[] data = {'零','壹','贰','叄','肆','伍','陆','柒','捌','玖'};
private static final char[] units = {'元','拾','佰','仟','万','拾','佰','仟','亿'};

/***
 * 金额转换为中文大写
 * @param money
 * @return
 */
public  String convert(Integer money){
    StringBuffer sbf = new StringBuffer();
    Integer uint = 0;
    while(money != 0){
        sbf.insert(0,units[uint++]);
        sbf.insert(0,data[money%10]);
        money = money/10;
    }
    //去零
    return sbf.toString().replaceAll("零[拾佰仟]","零").replaceAll("零+万","万").replaceAll("零+元","元").replaceAll("零+","零");

}
```

```java
@Override
public ResponceData<AnalyzeProductHandover> queryStorageProductHandover(Map<String, Object> map) {
    if (map.get("storageType") != null) {
        map.put("storageType", Integer.valueOf(map.get("storageType").toString()));
    }
    int count = mapper.queryProductHandoverCount(map);
    if (count <= 0) {
        return PageResult.result(null, 0);
    }
    List<AnalyzeProductHandover> details = mapper.queryProductHandover(map);
    for (AnalyzeProductHandover detail : details) {
        detail.setAlloyName(alloyService.get(detail.getAlloyId()).getName());
        detail.setStorageName(storageService.get(detail.getStorageId()).getName());
        detail.setSizeLengthSetName(ContextUtil.parseSize(detail.getSizeLengthSet()));
        detail.setProductNetWeight(detail.getProductWeight().subtract(detail.getStoneWeight()));
        BaseProductType productType = detail.getProductTypeId() != null ? baseProductTypeService.get(detail.getProductTypeId()) : null;
        detail.setProductTypeName(productType != null ? productType.getName() : null);
        switch (detail.getType()) {//操作选项类型：1、入库，2、出库，3、收货，4、出货，5、调入，6、调出，9、报损
            case 1://入库
                switch (detail.getBillType()) {
                    case 1:
                        detail.setSendName(ContextUtil.getStaff(detail.getSendId()).getDeptName() + "(" + ContextUtil.getStaff(detail.getSendId()).getName() + ")");
                        break;
                    case 2:
                        detail.setSendName(factoryService.get(detail.getSendId()).getClientShort());
                        break;
                }
                detail.setIncomeName(ContextUtil.getDeptName(detail.getIncomeId()));
                detail.setIncomeUserName(ContextUtil.getPersonName(detail.getIncomeUserId()));
                break;
            case 2://出库
                switch (detail.getBillType()) {
                    case 1:
                        detail.setIncomeName(ContextUtil.getStaff(detail.getIncomeId()).getDeptName() + "(" + ContextUtil.getStaff(detail.getIncomeId()).getName() + ")");
                        break;
                    case 2:
                        detail.setIncomeName(factoryService.get(detail.getIncomeId()).getClientShort());
                        break;
                }
                detail.setSendName(ContextUtil.getDeptName(detail.getSendId()));
                detail.setSendUserName(ContextUtil.getPersonName(detail.getSendUserId()));
                break;
            case 3://收货
            case 5://调入
                detail.setSendName(ContextUtil.getDeptName(detail.getSendId()));
                detail.setSendUserName(ContextUtil.getPersonName(detail.getSendUserId()));
                detail.setIncomeName(ContextUtil.getDeptName(detail.getIncomeId()));
                detail.setIncomeUserName(ContextUtil.getPersonName(detail.getIncomeUserId()));
                break;
            case 4://出货
                detail.setSendName(ContextUtil.getDeptName(detail.getSendId()));
                detail.setSendUserName(ContextUtil.getPersonName(detail.getSendUserId()));
                break;
            case 6://调出
                detail.setSendName(ContextUtil.getDeptName(detail.getSendId()));
                detail.setSendUserName(ContextUtil.getPersonName(detail.getSendUserId()));
                detail.setIncomeUserName(detail.getIncomeUserId() == null ? null : ContextUtil.getPersonName(detail.getIncomeUserId()));
                detail.setIncomeName(detail.getIncomeUserId() == null ? null : ContextUtil.getDeptName(ContextUtil.getStaff(detail.getIncomeUserId()).getDeptId()));
                break;
        }
    }
    return PageResult.result(details, count, mapper.queryProductHandoverSum(map));
```

```javascript
<select id="queryProductHandover" resultType="com.we7.erp.api.analyze.entity.AnalyzeProductHandover">
    select
    spb.bill_type, spb.storage_id, spb.send_bill_code, spb.income_bill_code, spb.operation_type_id as `type`,
    spb.income_date, spb.income_id, spb.income_user_id, spb.send_date, spb.send_id, spb.send_user_id, spd.memo,
    spd.product_id, spd.single_codes as singleCode, spd.product_amount, spd.product_weight, spd.stone_weight,
    spd.fitting_weight, spd.quote_price, spd.size_length_set, spd.single_id,spd.gross_weight as grossWeight,
    <if test="storageType==1"><!--公司存货仓-->
        oi.client_id, oi.id as orderId, oi.code as orderCode, oi.client_codes as clientCode,
        cpi.alloy_id, cpi.product_type_id, cpi.thumb_1, cpi.code as productCode, cc.client_short_cn as clientName,
        spd.quote_final_fee as quoteFinalFee
    </if>
    <if test="storageType==2"><!--客户成品仓-->
        oi.client_id, oi.id as orderId, oi.code as orderCode, oi.client_codes as clientCode, opi.alloy_id,
        opi.product_type_id, opi.thumb_1, opi.code as productCode, cc.client_short_cn as clientName,
        spb.ingredient_price ,spd.quote_final_fee as quoteFinalFee,pud.name_short as amountUnit,
        bp.name_cn as plating, oip.price as silverPrice
    </if>
    <include refid="storage_product_table"/>
    order by
    <if test="timeSort == 0">
        spd.id,
    </if>
    <if test="timeSort == 1">
        spd.id desc,
    </if>
    spb.operation_type_id
    <if test="limit > 0">
        limit #{startLimit}, #{limit}
    </if>
</select>
```

```java
<!--成品移交分析-->
<sql id="storage_product_table">
    from storage_product_bill spb
    join storage_product_detail spd on spd.bill_id = spb.id
    join storage st on st.id = spb.storage_id
    join hr_dept dept on st.company_id = dept.id
    <if test="storageType == 1"><!--公司存货仓-->
        join complex_product_info cpi on spd.product_id = cpi.id
        left join order_batch_product_single obps on obps.id = spd.single_id
        left join order_info oi on obps.order_id = oi.id
    </if>
    <if test="storageType == 2"><!--客户成品仓-->
        join order_product_info opi on spd.product_id = opi.id
        join order_info oi on opi.order_id = oi.id
        left join  order_ingredient_price oip on opi.order_id = oip.order_id
        join param_unit_detail pud on opi.metering_unit_id = pud.id
        join Base_Plating bp on opi.plating_id = bp.id
    </if>
    left join crm_client cc on cc.id=oi.client_id
    <where>
        spb.operation_type_id != 9<!--不查报损的数据-->queryProductHandover
        and if(spb.operation_type_id in (1,3,5),spb.income_bill_status,spb.send_bill_status) = 1
        <if test="storageType == 1"><!--公司存货仓-->
            and st.type = 9
            <if test="alloyId != null">
                and cpi.alloy_id = #{alloyId}
            </if>
            <if test="productId != null">
                and cpi.id = #{productId}
            </if>
        </if>
        <if test="storageType == 2"><!--客户成品仓-->
            and st.type = 11
            <if test="alloyId != null">
                and opi.alloy_id = #{alloyId}
            </if>
            <if test="productId != null">
                and opi.id = #{productId}
            </if>
            <if test="clientId != null">
                and spb.client_id = #{clientId}
            </if>
        </if>
        <if test="companyId != null">
            and dept.id = #{companyId}
        </if>
        <if test="type != null">
            and spb.operation_type_id = #{type}
            <if test="startDate != null">
                and if(spb.operation_type_id in (2,4,6), spb.send_date, spb.income_date) &gt;= #{startDate}
            </if>
            <if test="endDate != null">
                and if(spb.operation_type_id in (2,4,6), spb.send_date, spb.income_date) &lt;= #{endDate}
            </if>
        </if>
        <if test="type == null">
            <if test="startDate != null and endDate == null">
                and (spb.send_date &gt;= #{startDate}
                or spb.income_date &gt;= #{startDate})
            </if>
            <if test="startDate == null and endDate != null">
                and (spb.send_date &lt;= #{endDate}
                or spb.income_date &lt;= #{endDate})
            </if>
            <if test="startDate != null and endDate != null">
                and (spb.send_date &gt;= #{startDate} and spb.send_date &lt;= #{endDate}
                or spb.income_date &gt;= #{startDate} and spb.income_date &lt;= #{endDate})
            </if>
        </if>
        <if test="storageId != null">
            and spb.storage_id = #{storageId}
        </if>
        <if test="search != null">
            and
            <foreach collection="search" item="content" separator="and">
                (
                spb.send_bill_code like concat('%', #{content},'%') or
                spb.income_bill_code like concat('%', #{content},'%') or
                spd.single_codes like concat('%', #{content}, '%') or
                oi.code like concat('%', #{content},'%') or
                oi.client_codes like concat('%', #{content},'%') or
                cc.client_short_cn like concat('%',#{content},'%') or
                <if test="storageType == 1"><!--公司存货仓-->
                    cpi.code like concat('%', #{content},'%') or
                    obps.code like concat('%', #{content},'%') or
                    cpi.name_cn like concat('%', #{content},'%') or
                    cpi.name_en like concat('%', #{content},'%')
                </if>
                <if test="storageType == 2"><!--公司成品仓-->
                    opi.code like concat('%', #{content},'%') or
                    opi.name_cn like concat('%', #{content},'%') or
                    opi.name_en like concat('%', #{content},'%')
                </if>
                )
            </foreach>
        </if>
    </where>
</sql>
```

```java
 
```







1

```java
//查询产品订单信息
        OrderInfo info = orderInfoService.get(Long.valueOf(map.get("orderId").toString()));
        //查询订单的产品
        List<OrderProductInfo> productInfoList = orderProductInfoService.query(map);
        //返回数据实体
        List<Map<String, Object>> orderExcelVoList = new ArrayList<>();
        BigDecimal settlement = BigDecimal.ZERO;//汇率
        BigDecimal materWastage = BigDecimal.ZERO;//耗率
        BigDecimal materPrice = BigDecimal.ZERO;//主料单价
        for (OrderProductInfo productInfo : productInfoList) {
            Map<String, Object> quotationOrderExcelVo = new HashMap<>();
            OrderInside orderinside = orderInsideService.getByProductId(productInfo.getId(), null);
            //汇率计算
            OrderExchangeRate orderExchangeRate = new OrderExchangeRate();
            orderExchangeRate.setValuationId(Long.parseLong(map.get("currencyId").toString()));
            orderExchangeRate.setOrderId(productInfo.getOrderId());
            orderExchangeRate.setRateType(1);
            orderExchangeRate = orderExchangeRateService.getBean(orderExchangeRate);
            settlement = orderExchangeRate != null ? orderExchangeRate.getSettlement() : BigDecimal.ONE;//耗率
            ParamUnitDetail paramUnitDetail = unitDetailService.get(orderExchangeRate != null ? orderExchangeRate.getValuationId() : 0);
            //产品石头信息
            map.put("productId", productInfo.getId());
            List<OrderProductStone> productStoneList = orderProductStoneService.query(map);
            List<Map<String, Object>> stoneList = new ArrayList<>();
            BigDecimal stoneSumPrice = BigDecimal.ZERO;//总石值
            BigDecimal stoneSumFee = BigDecimal.ZERO;//总镶石工费
            for (OrderProductStone productStone : productStoneList) {
                Map<String, Object> stoneMap = new HashMap<>();
                FileStone fileStone = fileStoneService.get(productStone.getStoneId());
                OrderStonePrice stonePrice = orderStonePriceService.getByOrderAndStoneId(productStone.getOrderId(), productStone.getStoneId());
                ParamSettingDetail settingDetail = paramSettingDateilService.get(productStone.getSettingId());
                ParamSetting setting = paramSettingService.get(settingDetail.getSettingId());
                //石头基本
                ParamUnitDetail unitDetail = unitDetailService.get(productStone.getWeightId());
                stoneMap.put("weightName", unitDetail.getNameShort());//重量单位
                unitDetail = unitDetailService.get(productStone.getAmountId());
                stoneMap.put("amountName", unitDetail.getNameShort());//计量单位
                stoneMap.put("stoneName", fileStone.getName());//石头品名
                stoneMap.put("stoneWeight", productStone.getSingle());//重量
                stoneMap.put("stoneQuantity", productStone.getAmount());//粒数
                stoneMap.put("sellValuation", fileStone.getSellValuation());//计量方式
                stoneMap.put("sttingName", setting.getName() + "-" + settingDetail.getName());//镶法
                stoneMap.put("currency", paramUnitDetail != null ? paramUnitDetail.getNameShort() : "¥");//汇率符号
                stoneMap.put("notsumSellFee", orderExchangeRate == null ? productStone.getFee() : productStone.getFee().compareTo(BigDecimal.ZERO) != 0 ?
                        productStone.getFee().divide(orderExchangeRate.getSettlement(), 2, BigDecimal.ROUND_HALF_EVEN) : productStone.getFee());//镶工值
                stoneMap.put("notsumMaterial", orderExchangeRate == null ? stonePrice.getPrice() : stonePrice.getPrice().compareTo(BigDecimal.ZERO) != 0 ?
                        stonePrice.getPrice().divide(orderExchangeRate.getSettlement(), 2, BigDecimal.ROUND_HALF_EVEN) : stonePrice.getPrice());//石值
                stoneSumFee = stoneSumFee.add(productStone.getFee());
                stoneSumPrice = stoneSumPrice.add(productStone.getPrice());
                stoneList.add(stoneMap);
                quotationOrderExcelVo.put("stoneCode", quotationOrderExcelVo.get("stoneCode") == null ? fileStone.getName() + "-" + productStone.getAmount() + unitDetail.getNameShort()
                        : quotationOrderExcelVo.get("stoneCode").toString() + "\n" + fileStone.getName() + "-" + productStone.getAmount() + unitDetail.getNameShort());
            }
            //产品配件信息
            List<OrderProductFitting> productFittingList = orderProductFittingService.query(map);
            List<Map<String, Object>> fittingList = new ArrayList<>();
            BigDecimal fittingSumPrice = BigDecimal.ZERO;//总配件费
            for (OrderProductFitting productFitting : productFittingList) {
                Map<String, Object> fittingMap = new HashMap<>();
                FileFitting fileFitting = fileFittingService.get(productFitting.getFittingId());
                ParamUnitDetail unitDetail = unitDetailService.get(productFitting.getWeightId());
                fittingMap.put("weightName", unitDetail.getNameShort());//重量单位
                unitDetail = unitDetailService.get(productFitting.getAmountId());
                fittingMap.put("amountName", unitDetail.getNameShort());//计量单位
                fittingMap.put("fittingName", fileFitting.getName());//配件品名
                fittingMap.put("fittingWeight", productFitting.getSingle());//重量
                fittingMap.put("currency", paramUnitDetail != null ? paramUnitDetail.getNameShort() : "¥");//汇率符号
                fittingMap.put("needAmount", productFitting.getAmount());//数量
                fittingSumPrice = fittingSumPrice.add(productFitting.getFee());
                fittingMap.put("notsumSellFee", orderExchangeRate == null ? productFitting.getFee() : productFitting.getFee().compareTo(BigDecimal.ZERO) != 0 ?
                        productFitting.getFee().divide(orderExchangeRate.getSettlement(), 2, BigDecimal.ROUND_HALF_EVEN) : productFitting.getFee());//配件费
                fittingList.add(fittingMap);
            }
            ParamSysProcess processes = new ParamSysProcess();
            processes.setNameCn("倒模");
            Integer sysProcessId = paramSysProcessService.check(processes);
            List<ParamProcess> processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//倒模工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notcastprice", quotationOrderExcelVo.get("notcastprice") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notcastprice").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notcastprice", quotationOrderExcelVo.containsKey("notcastprice") ? quotationOrderExcelVo.get("notcastprice").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notcastprice", BigDecimal.ZERO);
            }
            processes.setNameCn("滚桶");
            sysProcessId = paramSysProcessService.check(processes);
            processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//滚桶工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notTumbling", quotationOrderExcelVo.get("notTumbling") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notTumbling").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notTumbling", quotationOrderExcelVo.containsKey("notTumbling") ? quotationOrderExcelVo.get("notTumbling").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notTumbling", BigDecimal.ZERO);
            }
            processes.setNameCn("抛光");
            sysProcessId = paramSysProcessService.check(processes);
            processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//抛光工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notpolishprice", quotationOrderExcelVo.get("notpolishprice") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notpolishprice").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notpolishprice", quotationOrderExcelVo.containsKey("notpolishprice") ? quotationOrderExcelVo.get("notpolishprice").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notpolishprice", BigDecimal.ZERO);
            }
            processes.setNameCn("执模");
            sysProcessId = paramSysProcessService.check(processes);
            processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//执模工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notsmithprice", quotationOrderExcelVo.get("notsmithprice") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notsmithprice").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notsmithprice", quotationOrderExcelVo.containsKey("notsmithprice") ? quotationOrderExcelVo.get("notsmithprice").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notsmithprice", BigDecimal.ZERO);
            }
            processes.setNameCn("电镀");
            sysProcessId = paramSysProcessService.check(processes);
            processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//电镀工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notplatingprice", quotationOrderExcelVo.get("notplatingprice") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notplatingprice").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notplatingprice", quotationOrderExcelVo.containsKey("notplatingprice") ? quotationOrderExcelVo.get("notplatingprice").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notplatingprice", BigDecimal.ZERO);
            }
            processes.setNameCn("烤漆/瓷");
            sysProcessId = paramSysProcessService.check(processes);
            processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//烤漆/瓷工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notpainPrice", quotationOrderExcelVo.get("notpainPrice") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notpainPrice").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notpainPrice", quotationOrderExcelVo.containsKey("notpainPrice") ? quotationOrderExcelVo.get("notpainPrice").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notpainPrice", BigDecimal.ZERO);
            }
            processes.setNameCn("烧焊");
            sysProcessId = paramSysProcessService.check(processes);
            processList = paramProcessService.getIdBySysProcessId(sysProcessId);
            if (processList.size() > 0) {//烧焊工费
                for (ParamProcess process : processList) {
                    OrderProductStep productStep = new OrderProductStep();
                    productStep.setOrderId(productInfo.getOrderId());
                    productStep.setProductId(productInfo.getId());
                    productStep.setStepId(process.getId());
                    productStep = orderProductStepService.getProductStep(productStep);
                    if (productStep != null) {
                        quotationOrderExcelVo.put("notwelding", quotationOrderExcelVo.get("notwelding") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notwelding").toString())).add(productStep.getFee()));
                        quotationOrderExcelVo.put("nototherPrice", quotationOrderExcelVo.get("nototherPrice") == null ? productStep.getFee() : BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("nototherPrice").toString())).add(productStep.getFee()));
                    } else {
                        quotationOrderExcelVo.put("notwelding", quotationOrderExcelVo.containsKey("notwelding") ? quotationOrderExcelVo.get("notwelding").toString() : BigDecimal.ZERO);
                        quotationOrderExcelVo.put("nototherPrice", quotationOrderExcelVo.containsKey("nototherPrice") ? quotationOrderExcelVo.get("nototherPrice").toString() : BigDecimal.ZERO);
                    }
                }
            } else {
                quotationOrderExcelVo.put("notwelding", BigDecimal.ZERO);
                quotationOrderExcelVo.put("nototherPrice", BigDecimal.ZERO);//其他费
            }
            OrderIngredientPrice orderIngredientPrice = orderIngredientPriceService.getOrderIngredPrice(null, productInfo.getOrderId(), productInfo.getAlloyId());
            if (materPrice.compareTo(BigDecimal.ZERO) == 0) {
                materPrice = orderIngredientPrice != null ? orderIngredientPrice.getPrice() : BigDecimal.ONE;
                materWastage = orderIngredientPrice != null ? orderIngredientPrice.getWastage() : BigDecimal.ONE;
            }
            FileIngredient fileIngredient = fileIngredientService.get(orderIngredientPrice != null ? orderIngredientPrice.getIngredientId() : 0);
            ParamAlloy paramAlloy = paramAlloyService.get(fileIngredient != null ? fileIngredient.getAlloyId() : 0);
            ParamAlloy alloy = paramAlloyService.get(productInfo.getAlloyId());
            BigDecimal valuationWeigth = BigDecimal.ZERO;
            OrderConfig orderConfig = new OrderConfig();
            orderConfig.setOrderId(productInfo.getOrderId());
            orderConfig = orderConfigService.getByOrder(orderConfig);
            if (orderConfig.getValuationWeightOption() == 0) {//取成品货重
                valuationWeigth = productInfo.getProductWeight();
            } else if (orderConfig.getValuationWeightOption() == 1) {//取成品净重
                valuationWeigth = productInfo.getProductNetWeight();
            } else if (orderConfig.getValuationWeightOption() == 2) {//取银版净重
                valuationWeigth = productInfo.getModelNetWeight();
            } else if (orderConfig.getValuationWeightOption() == 3) {//取倒模净重
                valuationWeigth = productInfo.getCastWeight();
            } else if (orderConfig.getValuationWeightOption() == 4) {//取主件净重
                valuationWeigth = productInfo.getMainPartsWeight();
            } else if (orderConfig.getValuationWeightOption() == 5) {//取折足净重
                valuationWeigth = productInfo.getProductNetWeight().multiply(alloy.getProductRatio());
            }
            if (paramAlloy != null && paramAlloy.getMaterialId().equals(alloy.getMaterialId())) {//银价/件=计价重量*银价*（1+耗率%）*
                BigDecimal bigDecimal = BigDecimal.valueOf(Double.parseDouble((valuationWeigth.multiply(orderIngredientPrice != null ? orderIngredientPrice.getPrice() : BigDecimal.ONE).multiply(BigDecimal.ONE.add(orderIngredientPrice != null ? orderIngredientPrice.getWastage() : BigDecimal.ZERO))).toPlainString()));
                quotationOrderExcelVo.put("notingredPrice", bigDecimal.compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : bigDecimal);
            } else {
                quotationOrderExcelVo.put("notingredPrice", valuationWeigth);
            }
            BigDecimal notingredPrice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notingredPrice").toString()));//银价/件
            //石头
            quotationOrderExcelVo.put("notstonePrice", stoneSumPrice);//总石值
            quotationOrderExcelVo.put("notsettingPrice", stoneSumFee);//总镶石工费
            //配件
            quotationOrderExcelVo.put("notfittingPrice", fittingSumPrice);//总配件费
            //工费值
            BigDecimal notTumbling = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notTumbling").toString()));//滚桶工费
            BigDecimal nototherPrice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("nototherPrice").toString()));//其他费
            BigDecimal notcastprice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notcastprice").toString()));//倒模工费
            BigDecimal notsmithprice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notsmithprice").toString()));//执模工费
            BigDecimal notpolishprice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notpolishprice").toString()));//抛光工费
            BigDecimal notplatingprice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notplatingprice").toString()));//电镀工费
            //工费/件=总石值+总镶石工费+总配件费+倒膜工费+滚筒工费+执膜工费+抛光工费+电镀费+其他费
            quotationOrderExcelVo.put("notquoteFinalFee", stoneSumPrice.add(stoneSumFee).add(fittingSumPrice).add(notcastprice).add(notTumbling).add(nototherPrice).add(notpolishprice).add(notplatingprice).add(notsmithprice));
            BigDecimal notquoteFinalFee = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notquoteFinalFee").toString()));//工费/件
            if ((notquoteFinalFee.compareTo(BigDecimal.ZERO) != 0 && productInfo.getProductWeight().compareTo(BigDecimal.ZERO) != 0) || (notquoteFinalFee.compareTo(BigDecimal.ZERO) == 0 && productInfo.getProductWeight().compareTo(BigDecimal.ZERO) != 0)) {
                quotationOrderExcelVo.put("notquoteG", notquoteFinalFee.divide(productInfo.getProductWeight(), 2, BigDecimal.ROUND_HALF_EVEN));//工费/克=（工费/件）/（ 成品重（成品货重））
            } else {
                quotationOrderExcelVo.put("notquoteG", BigDecimal.ZERO);
            }
            quotationOrderExcelVo.put("notquotePrice", notquoteFinalFee.add(notingredPrice));//报价/件=银价+工费/件
            BigDecimal notquoteG = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notquoteG").toString()));//工费/克
            BigDecimal notquotePrice = BigDecimal.valueOf(Double.parseDouble(quotationOrderExcelVo.get("notquotePrice").toString()));//报价/件
            //汇率计算
            if (orderExchangeRate != null) {
                quotationOrderExcelVo.put("notstonePrice", conversionCount(orderExchangeRate, stoneSumPrice));
                quotationOrderExcelVo.put("notsettingPrice", conversionCount(orderExchangeRate, stoneSumFee));
                quotationOrderExcelVo.put("notfittingPrice", conversionCount(orderExchangeRate, fittingSumPrice));
                quotationOrderExcelVo.put("notquoteG", conversionCount(orderExchangeRate, notquoteG));
                quotationOrderExcelVo.put("notTumbling", conversionCount(orderExchangeRate, notTumbling));
                quotationOrderExcelVo.put("notcastprice", conversionCount(orderExchangeRate, notcastprice));
                quotationOrderExcelVo.put("notquotePrice", conversionCount(orderExchangeRate, notquotePrice));
                quotationOrderExcelVo.put("notsmithprice", conversionCount(orderExchangeRate, notsmithprice));
                quotationOrderExcelVo.put("notingredPrice", conversionCount(orderExchangeRate, notingredPrice));
                quotationOrderExcelVo.put("notpolishprice", conversionCount(orderExchangeRate, notpolishprice));
                quotationOrderExcelVo.put("notplatingprice", conversionCount(orderExchangeRate, notplatingprice));
                quotationOrderExcelVo.put("notquoteFinalFee", conversionCount(orderExchangeRate, notquoteFinalFee));
            }
            //基础
            OrderInfo orderInfo = orderInfoService.get(productInfo.getOrderId());
            CostCrmPrice costCrmPrice = costCrmPriceService.getProductId(productInfo.getProductId(), orderInfo.getClientId());
            BasePaint paint = productInfo.getPaintId() != null ? basePaintService.get(productInfo.getPaintId()) : null;
            BasePlating plating = productInfo.getPlatingId() != null ? basePlatingService.get(productInfo.getPlatingId()) : null;
            BaseProductType type = productInfo.getProductTypeId() != null ? baseProductTypeService.get(productInfo.getProductTypeId()) : null;
            BaseProductType quality = productInfo.getQualityRequireId() != null ? baseProductQualityService.get(productInfo.getQualityRequireId()) : null;
            ParamMaterial material = alloy != null ? paramMaterialService.get(alloy.getMaterialId()) : null;
            quotationOrderExcelVo.put("stones", stoneList);//石头
            quotationOrderExcelVo.put("fittings", fittingList);//配件
            quotationOrderExcelVo.put("codes", productInfo.getCode());// 新款号
            quotationOrderExcelVo.put("memo", productInfo.getMemo());//产品备注
            quotationOrderExcelVo.put("clientCodes", info.getClientCodes());//客户单号
            quotationOrderExcelVo.put("processG", orderinside.getQuoteFee());// 报价工费
            quotationOrderExcelVo.put("ingredG", orderIngredientPrice.getPrice());//银价
            quotationOrderExcelVo.put("oldCodes", productInfo.getOriginalCode());//旧款号
            quotationOrderExcelVo.put("quotePrice", orderinside.getQuotePrice());//报价单价
            quotationOrderExcelVo.put("pain", paint != null ? paint.getName() : null);//烤漆
            quotationOrderExcelVo.put("alloy", alloy != null ? alloy.getName() : null);//成色
            quotationOrderExcelVo.put("productRatio", alloy.getProductRatio());//成品折足比率
            quotationOrderExcelVo.put("ringSize", orderinside.getSizeLengthSetName());//手寸/长度
            quotationOrderExcelVo.put("produceCruces", orderinside.getProduceCruces());//生产要点
            quotationOrderExcelVo.put("plating", plating != null ? plating.getName() : null);//电镀
            quotationOrderExcelVo.put("mater", material != null ? material.getName() : null);//材质
            quotationOrderExcelVo.put("quality", quality != null ? quality.getName() : null);//品质
            quotationOrderExcelVo.put("productType", type != null ? type.getName() : null);//产品类别
            quotationOrderExcelVo.put("currency", paramUnitDetail != null ? paramUnitDetail.getNameShort() : "¥");//汇率符号
            quotationOrderExcelVo.put("clientProductCode", costCrmPrice != null ? costCrmPrice.getClientCodes() : null);//客户款号
            quotationOrderExcelVo.put("orderAmount", orderinside.getAmount());//下单数量
            quotationOrderExcelVo.put("processWastage", orderIngredientPrice != null ? orderIngredientPrice.getWastage() : BigDecimal.ZERO);//产品耗率
            quotationOrderExcelVo.put("stoneWeight", productInfo.getStoneWeight());//石头重
            quotationOrderExcelVo.put("modelweight", productInfo.getModelNetWeight());//版重（成品净重）
            quotationOrderExcelVo.put("productWeight", productInfo.getProductWeight());//成品重（成品货重）
            quotationOrderExcelVo.put("productNetWeight", productInfo.getProductNetWeight());//银重（银版净重）
            quotationOrderExcelVo.put("image", StringUtils.isNotBlank(productInfo.getThumb1()) ? productInfo.getThumb1() : StringUtils.isNotBlank(productInfo.getThumb2()) ? productInfo.getThumb2() : productInfo.getThumb3());//图片
            orderExcelVoList.add(quotationOrderExcelVo);
        }
        Map<String, Object> quotationOrderMap = new HashMap<>();
        quotationOrderMap.put("codes", info.getCode());//订单号
        quotationOrderMap.put("settlement", settlement);// 汇率
        quotationOrderMap.put("materWastage", materWastage);//耗率
        quotationOrderMap.put("materPrice", materPrice);//主料单价
        quotationOrderMap.put("list", orderExcelVoList);//产品信息
        quotationOrderMap.put("clientCodes", info.getClientCodes());//客户单号
        quotationOrderMap.put("clientName", baseClientService.get(info.getClientId()).getName());//客户
        ParamUnitDetail paramUnitDetail = unitDetailService.get(Long.parseLong(map.get("currencyId").toString()));
        quotationOrderMap.put("exchange", paramUnitDetail != null ? paramUnitDetail.getName() : "人民币");//币种
        quotationOrderMap.put("offerDate", info.getOfferDate() != null ? DateHelp.formats(info.getShipmentDate()) : null);//报价日期
        return new Object[]{quotationOrderMap};
```























```
//查询产品订单信息
//        OrderInfo info = orderInfoService.get(Long.valueOf(map.get("orderId").toString()));
//        //查询订单的产品
//        List<OrderProductInfo> productInfoList = orderProductInfoService.query(map);
//        //返回数据实体
//        List<Map<String, Object>> orderExcelVoList = new ArrayList<>();
//        BigDecimal materPrice = BigDecimal.ZERO;//主料单价
//        for (OrderProductInfo productInfo : productInfoList) {
//            Map<String, Object> quotationOrderExcelVo = new HashMap<>();
//            OrderInside orderinside = orderInsideService.getByProductId(productInfo.getId(), null);
//            //产品石头信息
//            map.put("productId", productInfo.getId());
//            List<OrderProductStone> productStoneList = orderProductStoneService.query(map);
//            List<Map<String, Object>> stoneList = new ArrayList<>();
//            BigDecimal stoneSumPrice = BigDecimal.ZERO;//总石值
//            BigDecimal stoneSumFee = BigDecimal.ZERO;//总镶石工费
//            for (OrderProductStone productStone : productStoneList) {
//                Map<String, Object> stoneMap = new HashMap<>();
//                FileStone fileStone = fileStoneService.get(productStone.getStoneId());
//                OrderStonePrice stonePrice = orderStonePriceService.getByOrderAndStoneId(productStone.getOrderId(), productStone.getStoneId());
//                ParamSettingDetail settingDetail = paramSettingDateilService.get(productStone.getSettingId());
//                ParamSetting setting = paramSettingService.get(settingDetail.getSettingId());
//                //石头基本
//                ParamUnitDetail unitDetail = unitDetailService.get(productStone.getWeightId());
//                stoneMap.put("weightName", unitDetail.getNameShort());//重量单位
//                unitDetail = unitDetailService.get(productStone.getAmountId());
//                stoneMap.put("amountName", unitDetail.getNameShort());//计量单位
//                stoneMap.put("stoneName", fileStone.getName());//石头品名
//                stoneMap.put("stoneWeight", productStone.getSingle());//重量
//                stoneMap.put("stoneQuantity", productStone.getAmount());//粒数
//                stoneMap.put("sellValuation", fileStone.getSellValuation());//计量方式
//                stoneMap.put("sttingName", setting.getName() + "-" + settingDetail.getName());//镶法
//                stoneMap.put("currency", paramUnitDetail != null ? paramUnitDetail.getNameShort() : "¥");//汇率符号
//                stoneMap.put("notsumSellFee", orderExchangeRate == null ? productStone.getFee() : productStone.getFee().compareTo(BigDecimal.ZERO) != 0 ?
//                        productStone.getFee().divide(orderExchangeRate.getSettlement(), 2, BigDecimal.ROUND_HALF_EVEN) : productStone.getFee());//镶工值
//                stoneMap.put("notsumMaterial", orderExchangeRate == null ? stonePrice.getPrice() : stonePrice.getPrice().compareTo(BigDecimal.ZERO) != 0 ?
//                        stonePrice.getPrice().divide(orderExchangeRate.getSettlement(), 2, BigDecimal.ROUND_HALF_EVEN) : stonePrice.getPrice());//石值
//                stoneSumFee = stoneSumFee.add(productStone.getFee());
//                stoneSumPrice = stoneSumPrice.add(productStone.getPrice());
//                stoneList.add(stoneMap);
//                quotationOrderExcelVo.put("stoneCode", quotationOrderExcelVo.get("stoneCode") == null ? fileStone.getName() + "-" + productStone.getAmount() + unitDetail.getNameShort()
//                        : quotationOrderExcelVo.get("stoneCode").toString() + "\n" + fileStone.getName() + "-" + productStone.getAmount() + unitDetail.getNameShort());
//            }
//            //产品配件信息
//            List<OrderProductFitting> productFittingList = orderProductFittingService.query(map);
//            List<Map<String, Object>> fittingList = new ArrayList<>();
//            BigDecimal fittingSumPrice = BigDecimal.ZERO;//总配件费
//            for (OrderProductFitting productFitting : productFittingList) {
//                Map<String, Object> fittingMap = new HashMap<>();
//                FileFitting fileFitting = fileFittingService.get(productFitting.getFittingId());
//                ParamUnitDetail unitDetail = unitDetailService.get(productFitting.getWeightId());
//                fittingMap.put("weightName", unitDetail.getNameShort());//重量单位
//                unitDetail = unitDetailService.get(productFitting.getAmountId());
//                fittingMap.put("amountName", unitDetail.getNameShort());//计量单位
//                fittingMap.put("fittingName", fileFitting.getName());//配件品名
//                fittingMap.put("fittingWeight", productFitting.getSingle());//重量
//                fittingMap.put("currency", paramUnitDetail != null ? paramUnitDetail.getNameShort() : "¥");//汇率符号
//                fittingMap.put("needAmount", productFitting.getAmount());//数量
//                fittingSumPrice = fittingSumPrice.add(productFitting.getFee());
//                fittingMap.put("notsumSellFee", orderExchangeRate == null ? productFitting.getFee() : productFitting.getFee().compareTo(BigDecimal.ZERO) != 0 ?
//                        productFitting.getFee().divide(orderExchangeRate.getSettlement(), 2, BigDecimal.ROUND_HALF_EVEN) : productFitting.getFee());//配件费
//                fittingList.add(fittingMap);
//            }
//        }
```





时间转换:DateHelp.formatToYearMonth(new SimpleDateFormat("yyyy-MM-dd").parse(map.get("startDate").toString()))

analyze.setSendDateStr(analyze.getSendDate() !=null ? new SimpleDateFormat("yyyy-MM-dd").format(analyze.getSendDate()) : null);
产品跟踪。产品使用记录，下单和生产

storage_stone_detail

判断是那个excel表格

 if (map.get("path").toString().contains("DYShipmentProductSummary")) {

contains（）,该方法是判断字符串中是否有子字符串。如果有则返回true，如果没有则返回false。

小数点截取 BigDecimal.setScale(1,BigDecimal.Round_DOWN)直接删除多余的小数位



```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.5.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
<groupId>com</groupId>
<artifactId>blog</artifactId>
<version>0.0.1-SNAPSHOT</version>
<name>blog1</name>
<description>Demo project for Spring Boot</description>

<properties>
    <java.version>1.8</java.version>
</properties>

<dependencies>
    <!--这三个jar包作用是将markdown格式转成html格式-->
    <dependency>
        <groupId>com.atlassian.commonmark</groupId>
        <artifactId>commonmark</artifactId>
        <version>0.10.0</version>
    </dependency>

    <dependency>
        <groupId>com.atlassian.commonmark</groupId>
        <artifactId>commonmark-ext-heading-anchor</artifactId>
        <version>0.10.0</version>
    </dependency>
    <dependency>
        <groupId>com.atlassian.commonmark</groupId>
        <artifactId>commonmark-ext-gfm-tables</artifactId>
        <version>0.10.0</version>
    </dependency>

    <dependency>
        <groupId>com.github.pagehelper</groupId>
        <artifactId>pagehelper-spring-boot-starter</artifactId>
        <version>1.2.13</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.1.1</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>

</project>
```





```java
//判断胶模仓是否允许为负
        if (!config.getIsMouldStorageNegative()) {
```



```java
List<AnalyzeProductHandover> data = this.queryStorageProductHandover(map).getData() != null ? (List<AnalyzeProductHandover>) this.queryStorageProductHandover(map).getData() : new ArrayList<>();
        List<AnalyzeProductHandover> handovers = data.stream().collect(
                Collectors.collectingAndThen(Collectors.toCollection(() -> new TreeSet<>(Comparator.comparing(o -> o.getSendBillCode()))),
                        ArrayList::new));
        List<Map<String, Object>> exportLists = new ArrayList<>();
        for (AnalyzeProductHandover handover : handovers) {
            List<Map<String, Object>> detail = new ArrayList<>();
            Map<String, Object> exportMap = new HashMap<>();
            for (AnalyzeProductHandover analyze : data) {
                Map<String, Object> detailMap = new HashMap<>();
                analyze.setSendDateStr(analyze.getSendDate() == null ? null : DateHelp.formats(analyze.getSendDate()));
                analyze.setTotalPrice(analyze.getProcessFinalFeeTotal().multiply(analyze.getProductAmount()));
                map.put("productId", analyze.getOriginalProductId());//原始订单产品编号
                BigDecimal fees = BigDecimal.ZERO;  //3d工序价格
                Map<String, Object> map1 = new HashMap<String, Object>();
                if (map.get("fileName").equals("3d建模出货明细表")) {
                    map1.put("stepId", 2L);
                    map1.put("productId", map.get("productId"));
                    map1.put("isCompany", true);
                    List<CostStepPrice> stepPrices = stepPriceService.query(map1);
                    CostStepPrice costStepPrice = stepPrices.size() == 0 ? new CostStepPrice() : stepPrices.get(0);
                    fees = fees.add(costStepPrice.getSellFee());
                } else if (map.get("fileName").equals("手绘出货明细表")) {
                    map1.put("stepId", 1L); //手绘工序id
                    map1.put("productId", map.get("productId"));
                    map1.put("isCompany", true);
                    List<CostStepPrice> stepPrices = stepPriceService.query(map1);
                    CostStepPrice costStepPrice = stepPrices.size() == 0 ? new CostStepPrice() : stepPrices.get(0);
                    fees = fees.add(costStepPrice.getSellFee());
                }
                analyze.setQuoteFinalFeeSum(fees.multiply(analyze.getProductAmount()));
                analyze.setQuoteFinalFee(fees);
                if (handover.getSendBillCode().equals(analyze.getSendBillCode())) {
                    detailMap.put("productCode", analyze.getProductCode());
                    detailMap.put("thumb1", analyze.getThumb1());
                    detailMap.put("quoteFinalFee", analyze.getQuoteFinalFee());
                    detailMap.put("memo", analyze.getMemo());
                    detailMap.put("productAmount", analyze.getProductAmount());
                    detail.add(detailMap);
                }
            }
            exportMap.put("sendBillCode", handover.getSendBillCode());
            exportMap.put("sendDateStr", handover.getSendDateStr());
            exportMap.put("clientName", handover.getClientName());
            exportMap.put("detail", detail);
            exportLists.add(exportMap);
        }
        map.put("list", exportLists);
        map.put("date", map.containsKey("startDate") ? DateHelp.formatToYearMonth(new SimpleDateFormat("yyyy-MM-dd").parse(map.get("startDate").toString())) : null);//日期
        map.put("clientName", map.get("clientId") != null ? clientService.get(Long.valueOf(map.get("clientId").toString())).getName() : "");
        //汇总数据
        Map<String, Object> tmap = new HashMap<>();
        if (map.get("path").toString().contains("3D建模表格格式")) {
            //分组算汇总
            Map<String, List<AnalyzeProductHandover>> listMap = data.stream().collect(Collectors.groupingBy(d -> d.getClientId() + "-" + d.getSendBillCode()));
            List<Map<String, Object>> totalList = listMap.values().stream().map(d -> {
                Map<String, Object> m = new HashMap<>();
                m.put("sendDate", d.get(0).getSendDate());
                m.put("sendDateStr", DateHelp.formats(d.get(0).getSendDate()));
                m.put("sendBillCode", d.get(0).getSendBillCode());
                m.put("clientName", d.get(0).getClientName());
                m.put("productAmount", d.stream().map(AnalyzeProductHandover::getProductAmount).reduce(BigDecimal.ZERO, BigDecimal::add));
                m.put("productWeight", d.stream().map(AnalyzeProductHandover::getProductWeight).reduce(BigDecimal.ZERO, BigDecimal::add));
                m.put("quoteFinalFeeSum", d.stream().map(AnalyzeProductHandover::getQuoteFinalFeeSum).reduce(BigDecimal.ZERO, BigDecimal::add));
                m.put("ingredientPrice", d.get(0).getIngredientPrice());
                return m;
            }).collect(Collectors.toList());
            totalList.sort((s1, s2) -> ((Date) s1.get("sendDate")).compareTo((Date) s2.get("sendDate")));
            tmap.put("clientName", map.get("clientId") != null ? clientService.get(Long.valueOf(map.get("clientId").toString())).getName() : "");
            tmap.put("list", totalList);
            tmap.put("date", map.get("date"));
            return new Object[]{tmap, map};
        } else {
            return new Object[]{map};
        }
```

4

```java
StorageIngredientBill bill = billService.get(detail.getBillId());
        if (bill == null) {
            throw new BusinessException(ExceptionMessage.getSelectNot());
        }
        StorageIngredientInventory inventory = inventoryService.getByIngredient(bill.getStorageId(),detail.getIngredientId());
        if (detail.getSarkCodes() == null) { //如果没填柜子编号以库存为准
            detail.setSarkCodes(inventory!=null ?inventory.getSarkCodes() :null);
        }
        //如果是入
        if (StringHelp.ifExists("1,3", bill.getOperationTypeId().toString())) {
            if (bill.getIncomeBillStatus() == 1) {
                throw new BusinessException(ExceptionMessage.getFinishNotUpdate());
            }
            //新建入库则发出默认为已确认
            detail.setSendSingleStatus(1);
        } else {
            if (bill.getSendBillStatus() == 1) {
                throw new BusinessException(ExceptionMessage.getFinishNotUpdate());
            }
        }

        //判断主料是否存在
        FileIngredient ingredient = ingredientService.get(detail.getIngredientId());
        if (ingredient == null) {
            throw new BusinessException(ExceptionMessage.getSelectNot());
        }

        //如果不在本身权限内则不能添加
        Map<String, Object> pares = new HashMap<>();
        pares.put("staffId", ThreadMapUtil.getStaffId());
        List<ParamStaffConfig> configs = staffConfigService.query(pares);
        if (configs != null && configs.size() > 0 && !StringHelp.ifExists(configs.get(0).getMaterialIds(), ContextUtil.getAlloyByMaterial(ingredient.getAlloyId()))) {
            throw new BusinessException(ExceptionMessage.getAuthNot());
        }
        ParamIngredientType type = typeService.get(ingredient.getIngredientClassify());
        detail.setWeightUnitId(type.getWeightUnit());
        if (detail.getPrice().compareTo(BigDecimal.ZERO) == 0) {
            Integer id = ingredientBillService.check();
            if (id != null && id != 0) {
                CostIngredientPrice price = new CostIngredientPrice();
                price.setBillId(id.longValue());
                price.setIngredientId(detail.getIngredientId());
                price = ingredientPriceService.getByCondition(price);
                detail.setPrice(price == null || price.getPrice() == null ? BigDecimal.ZERO : price.getPrice());
            }
        }
        return super.insert(detail);
```

