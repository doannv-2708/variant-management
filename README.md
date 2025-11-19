# 📌 Variant Management trong OpenUI5

Variant Management là tính năng quan trọng khi bạn muốn cho phép người dùng **lưu – tải lại – chia sẻ** các cấu hình giao diện như:

* Bộ filter (SmartFilterBar / FilterBar)
* Cột hiển thị, sort, group (SmartTable / Table)
* Layout, view state...

Mỗi **variant** chính là một snapshot của trạng thái UI → giúp người dùng không phải thiết lập lại thủ công.

---

# 1. Các cách triển khai Variant Management trong OpenUI5

Có 2 hướng tiếp cận:

---

## ✅ **Cách 1 – Dùng SmartVariantManagement / Smart Controls (khuyến nghị)**

OpenUI5 cung cấp toàn bộ UI + backend Flexibility Service.
👉 **Không cần tạo bảng Z, không cần viết OData thủ công.**

### 📄 XML View mẫu

```xml
<mvc:View controllerName="smarttable.controller.View1"
    xmlns:mvc="sap.ui.core.mvc" displayBlock="true"
    xmlns:smartTable="sap.ui.comp.smarttable"
    xmlns:smartFilterBar="sap.ui.comp.smartfilterbar"
    xmlns="sap.m">

    <Page id="page" title="{i18n>title}">
        <content>

            <smartFilterBar:SmartFilterBar 
                id="smartFilterBar"
                entitySet="Products"
                persistencyKey="SmartFilter_Explored">

                <smartFilterBar:controlConfiguration>
                    <smartFilterBar:ControlConfiguration key="ID" visibleInAdvancedArea="true">
                        <smartFilterBar:defaultFilterValues>
                            <smartFilterBar:SelectOption low="1" />
                        </smartFilterBar:defaultFilterValues>
                    </smartFilterBar:ControlConfiguration>

                    <smartFilterBar:ControlConfiguration key="Name" visibleInAdvancedArea="true">
                        <smartFilterBar:defaultFilterValues>
                            <smartFilterBar:SelectOption low="Milk" />
                        </smartFilterBar:defaultFilterValues>
                    </smartFilterBar:ControlConfiguration>
                </smartFilterBar:controlConfiguration>

            </smartFilterBar:SmartFilterBar>

            <smartTable:SmartTable 
                smartFilterId="smartFilterBar"
                id="smartTable"
                initiallyVisibleFields="ID,Name,Description,ReleaseDate"
                entitySet="Products"
                tableType="ResponsiveTable"
                enableExport="true"
                useVariantManagement="true"
                useTablePersonalisation="true"
                header="Product Table"
                showRowCount="true"
                persistencyKey="SmartTable_Explored"
                enableAutoBinding="true"
                enableAutoColumnWidth="false"
                class="sapUiResponsiveContentPadding">
            </smartTable:SmartTable>

        </content>
    </Page>
</mvc:View>
```

### 🔍 Giải thích nhanh

| Thuộc tính                    | Ý nghĩa                                                 |
| ----------------------------- | ------------------------------------------------------- |
| `entitySet="Products"`        | SmartFilterBar tự sinh filter controls dựa vào metadata |
| `persistencyKey="..."`        | Key để OpenUI5 lưu variant                              |
| `useVariantManagement="true"` | Bật lưu variant trên SmartTable                         |

---

## ❗ Cách 2 – Tự xây dựng Variant Management thủ công

Bạn phải:

* Tạo **Z-table** để lưu variants
* Tạo **OData service (CRUD)**: create, read, update, delete
* Tự viết toàn bộ logic JavaScript

→ Mất rất nhiều effort nhưng phù hợp nếu bạn muốn quản lý variant theo business riêng.

---

# 2. Controller (Tự xây dựng Variant Management)

## 📌 Khởi tạo – load danh sách variant từ OData

```javascript
sap.ui.define([
    "sap/ui/core/mvc/Controller",
    "sap/ui/model/Filter",
    "sap/ui/model/FilterOperator"
], function (Controller) {
    "use strict";
    return Controller.extend("project1.controller.View1", {

        onInit() {
            this._oDataModel = new sap.ui.model.odata.ODataModel("/sap/opu/odata/sap/Z_VARIANTS_API");

            var oModel = new sap.ui.model.json.JSONModel({
                VariantSet: []
            });
            this.getView().setModel(oModel);

            var oModelSelection = new sap.ui.model.json.JSONModel({
                first_profile: "",
                second_profile: "",
                critical: false
            });
            oModelSelection.setDefaultBindingMode(sap.ui.model.BindingMode.OneWay);
            this.getView().setModel(oModelSelection, "selection");

            this._oDataModel.read("/ZI_VARIANTS", {
                success: (oData) => {
                    oModel.setProperty("/VariantSet", oData.results);
                },
                error: () => {
                    alert("Service Failed");
                }
            });
        },
```

---

## 📌 Lưu Variant (Save / Save As)

```javascript
onSave: function (oEvent) {
  var params = oEvent.getParameters();
  var oDataModel = new sap.ui.model.odata.ODataModel("/sap/opu/odata/sap/Z_VARIANTS_API");

  if (params.overwrite) {
    var parametersValue = this.getParametersValue();
    var selectedKey = oEvent.getSource().getSelectionKey();
    var bindingPath = oEvent.getSource().getItemByKey(selectedKey).getBindingContext().getPath();
    var modelData = sap.ui.getCore().byId("app").getModel().getProperty(bindingPath);

    var save = {
      first_profile: parametersValue.firstProfile,
      second_profile: parametersValue.secondProfile,
      critical: parametersValue.critical,
      var_key: modelData.var_key,
      var_name: modelData.var_name,
    };

    $.extend(modelData, save);
    sap.ui.getCore().byId("app").getModel().refresh();

    oDataModel.update("/ZI_VARIANTS('" + save.var_key + "')", save);

  } else {
    var parametersValue = this.getParametersValue();
    var newEntry = {
      var_name: params.name,
      var_key: params.key,
      first_profile: parametersValue.firstProfile,
      second_profile: parametersValue.secondProfile,
      critical: parametersValue.critical,
    };

    oDataModel.create("/ZI_VARIANTS", newEntry, null, (oData) => {
      var Data = sap.ui.getCore().byId("app").getModel().getData().VariantSet;
      Data.push(newEntry);
      sap.ui.getCore().byId("app").getModel().refresh();
    });
  }
},
```

---

## 📌 Quản lý variant (Rename / Delete)

Có xử lý **ETag**, tránh lỗi CSRF/412 Precondition Failed.

```javascript
onManage: function (oEvent) {
    var oDataModel = this._oDataModel;
    var oJsonModel = this.getView().getModel();
    var aVariants = oJsonModel.getProperty("/VariantSet");

    var params = oEvent.getParameters();
    var renamed = params.renamed;
    var deleted = params.deleted;

    // Rename
    if (renamed && renamed.length > 0) {
        renamed.forEach(item => {
            var oVariantData = aVariants.find(v => v.var_key === item.key);
            if (!oVariantData) return;

            var payload = { var_name: item.name };
            var mParams = {
                eTag: oVariantData.last_changed_at,
                success: () => {
                    oVariantData.var_name = item.name;
                    oJsonModel.refresh();
                }
            };

            oDataModel.update("/ZI_VARIANTS('" + item.key + "')", payload, mParams);
        });
    }

    // Delete
    if (deleted && deleted.length > 0) {
        deleted.forEach(keyToRemove => {
            var oVariantData = aVariants.find(v => v.var_key === keyToRemove);
            var iIndex = aVariants.findIndex(v => v.var_key === keyToRemove);

            if (!oVariantData) return;

            var mParams = {
                eTag: oVariantData.last_changed_at,
                success: () => {
                    aVariants.splice(iIndex, 1);
                    oJsonModel.refresh();
                }
            };

            oDataModel.remove("/ZI_VARIANTS('" + keyToRemove + "')", mParams);
        });
    }
},
```

---

## 📌 Chọn variant (Select)

```javascript
onSelect: function (oEvent) {
    var selectedKey = oEvent.getSource().getSelectionKey();
    var oSelectionModel = this.getView().getModel("selection");

    if (selectedKey === "*standard*") {
        oSelectionModel.setData({
            FIRST_PROFILE: "",
            SECOND_PROFILE: "",
            CRITICAL: false
        });

    } else {
        var oMainModel = this.getView().getModel();
        var bindingPath = oEvent.getSource().getItemByKey(selectedKey).getBindingContext().getPath();
        var oVariantData = oMainModel.getProperty(bindingPath);

        var oCopiedData = jQuery.extend({}, oVariantData);
        oCopiedData.critical = (oCopiedData.critical === "true" || oCopiedData.critical === true);

        oSelectionModel.setData(oCopiedData);
    }
},
```

---

# 3. View (FilterBar + VariantManagement)

```xml
<mvc:View
    controllerName="project1.controller.View1"
    displayBlock="true"
    xmlns:mvc="sap.ui.core.mvc"
    xmlns="sap.m"
    xmlns:fb="sap.ui.comp.filterbar"
    xmlns:v="sap.ui.comp.variants">

    <App id="app">
        <Page title="Variant Management" showNavButton="true">

            <content>

                <v:VariantManagement
                    id="vm"
                    select="onSelect"
                    save="onSave"
                    manage="onManage"
                    showExecuteOnSelection="true"
                    variantItems="{/VariantSet}"
                    useShellPersonalization="true">

                    <v:variantItems>
                        <v:VariantItem text="{var_name}" key="{var_key}" />
                    </v:variantItems>

                </v:VariantManagement>

                <fb:FilterBar id="fb" search="onGo">

                    <fb:filterItems>

                        <fb:FilterItem name="A" label="FirstProfile">
                            <fb:control>
                                <Input value="{selection>/first_profile}" />
                            </fb:control>
                        </fb:FilterItem>

                        <fb:FilterItem name="B" label="Second Profile">
                            <fb:control>
                                <Input value="{selection>/second_profile}" />
                            </fb:control>
                        </fb:FilterItem>

                        <fb:FilterItem name="C">
                            <fb:control>
                                <CheckBox selected="{selection>/critical}" text="Critical" />
                            </fb:control>
                        </fb:FilterItem>

                    </fb:filterItems>

                </fb:FilterBar>

            </content>
        </Page>
    </App>
</mvc:View>
```

---

# 4. So sánh SmartVariantManagement vs. Làm thủ công

| Tiêu chí               | Smart Variant Management (OpenUI5) | Tự xây dựng thủ công         |
| ---------------------- | ---------------------------------- | ---------------------------- |
| UI                     | Có sẵn toàn bộ UI Fiori            | Tự thiết kế UI               |
| Lưu trữ                | Flexibility Service tự lưu         | Phải tạo bảng Z + OData CRUD |
| Frontend               | Tự động binding                    | Phải viết JS thủ công        |
| State tracking         | Có sẵn “dirty state”               | Tự code so sánh              |
| Public/Private variant | Có built-in                        | Phải tự làm                  |
| Effort                 | Rất thấp                           | Rất cao                      |

---

# ✔ Kết luận

* Nếu app dùng **SmartFilterBar / SmartTable** → **NÊN dùng SmartVariantManagement**.
* Chỉ nên tự làm nếu có yêu cầu business đặc thù (quyền, phân hệ, custom field…).

