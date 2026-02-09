# Account

## Overview
- **Row Count**: 54,719
- **Primary Key**: ID

## Columns

| Column Name | Data Type | Max Length | Nullable |
|-------------|-----------|-----------|----------|
| ID | int | - | NO |
| StoreID | int | - | NO |
| ClassTypeID | int | - | NO |
| ModifiedByUser | nvarchar | 25 | YES |
| ModifiedByComputer | nvarchar | 25 | YES |
| ModifiedDate | datetime | - | YES |
| SeqID | int | - | YES |
| IsSystem | bit | - | YES |
| IsActive | bit | - | YES |
| Notes | text | 2147483647 | YES |
| CompanyName | nvarchar | 50 | YES |
| AccountNumber | int | - | YES |
| ParentID | int | - | YES |
| Department | nvarchar | 25 | YES |
| DateCreated | datetime | - | YES |
| DateImported | datetime | - | YES |
| ImportBatch | nvarchar | 15 | YES |
| AccountingContactID | int | - | YES |
| PrimaryContactID | int | - | YES |
| BillingAddressID | int | - | YES |
| ShippingAddressID | int | - | YES |
| MainPhoneNumberID | int | - | YES |
| MainFaxNumberID | int | - | YES |
| Flags | text | 2147483647 | YES |
| Keywords | text | 2147483647 | YES |
| TaxNumber | nvarchar | 25 | YES |
| TaxNumberExpDate | datetime | - | YES |
| TaxClassID | int | - | YES |
| WebAddress | nvarchar | 50 | YES |
| IsProspect | bit | - | YES |
| TaxExempt | bit | - | YES |
| HasCreditAccount | bit | - | YES |
| CreditApprovalDate | datetime | - | YES |
| CreditLimit | decimal | - | YES |
| CreditBalance | decimal | - | YES |
| DefaultPaymentExpDate | datetime | - | YES |
| DefaultPaymentTrackingNumber | nvarchar | 25 | YES |
| DefaultPaymentNameOnCard | nvarchar | 25 | YES |
| DefaultPaymentTypeID | int | - | YES |
| PricingPlanTypeID | int | - | YES |
| PaymentTermsID | int | - | YES |
| DiscountLevel | float | - | YES |
| PricingLevel | float | - | YES |
| PONumberRequired | bit | - | YES |
| IndustryID | int | - | YES |
| OriginID | int | - | YES |
| Marketing3ID | int | - | YES |
| SalesPersonID1 | int | - | YES |
| SalesPersonID2 | int | - | YES |
| SalesPersonID3 | int | - | YES |
| TaxExemptExpDate | datetime | - | YES |
| CreditNumber | nvarchar | 25 | YES |
| PricingLevelID | int | - | YES |
| PromotionID | int | - | YES |
| UseTaxLookup | bit | - | YES |
| HasServiceContract | bit | - | YES |
| ServiceContractStartDate | datetime | - | YES |
| ServiceContractExpDate | datetime | - | YES |
| ServiceContractTypeID | int | - | YES |
| ServiceContractNotes | text | 2147483647 | YES |
| DivisionID | int | - | YES |
| RegionID | int | - | YES |
| PONumber | varchar | 25 | YES |
| PrimaryNumber | varchar | 75 | YES |
| PriNumberTypeID | int | - | YES |
| PriNumberTypeText | varchar | 50 | YES |
| SecondaryNumber | varchar | 75 | YES |
| SecNumberTypeID | int | - | YES |
| SecNumberTypeText | varchar | 50 | YES |
| IsClient | bit | - | YES |
| IsVendor | bit | - | YES |
| IsPersonal | bit | - | YES |
| Is1099Vendor | bit | - | YES |
| VendorPaymentTermsID | int | - | YES |
| MyAccountNumber | varchar | 50 | YES |
| DefaultShipMethodID | int | - | YES |
| ThirdNumber | varchar | 75 | YES |
| ThirdNumberTypeID | int | - | YES |
| ThirdNumberTypeText | varchar | 50 | YES |
| IsFullyTaxExempt | bit | - | YES |
| VendorCreditBalance | float | - | YES |
| StageID | int | - | YES |
| StageClassTypeID | int | - | YES |
| StageActivityID | int | - | YES |
| StageActivityClassTypeID | int | - | YES |
| LicenseKey | uniqueidentifier | - | YES |
| DefaultPaymentMethodID | int | - | YES |
| PriContactNumber | int | - | YES |
| EntityUseCode | varchar | 25 | YES |
| EntityUseCodeName | varchar | 25 | YES |
| BillingFormattedNumber | varchar | 75 | YES |
| BillingContactNumber | varchar | 75 | YES |
| PrimaryContactNumber | varchar | 75 | YES |

## Foreign Keys

| FK Name | Column | References |
|---------|--------|------------|
| FK_Account_DivisionData | DivisionID | DivisionData.ID |
| FK_Account_PricingLevel | PricingLevelID | PricingLevel.ID |
| FK_Account_Promotion | PromotionID | Promotion.ID |
| FK_Account_VendorPaymentTerms | VendorPaymentTermsID | PaymentTerms.ID |

## Sample Data
- **ID**: 10
- **StoreID**: 100
- **ClassTypeID**: 2000
- **ModifiedByUser**: null
- **ModifiedByComputer**: null
- **ModifiedDate**: 2025-11-12T16:22:46.897Z
- **SeqID**: 2
- **IsSystem**: false
- **IsActive**: true
- **Notes**: null
- **CompanyName**: Walk-In
- **AccountNumber**: 0
- **ParentID**: null
- **Department**: null
- **DateCreated**: null
- **DateImported**: null
- **ImportBatch**: null
- **AccountingContactID**: 10
- **PrimaryContactID**: 10
- **BillingAddressID**: 3706
- **ShippingAddressID**: 3706
- **MainPhoneNumberID**: 26575
- **MainFaxNumberID**: 26576
- **Flags**: null
- **Keywords**: null
- **TaxNumber**: null
- **TaxNumberExpDate**: null
- **TaxClassID**: 10012
- **WebAddress**: null
- **IsProspect**: false
- **TaxExempt**: false
- **HasCreditAccount**: false
- **CreditApprovalDate**: null
- **CreditLimit**: 0
- **CreditBalance**: 0
- **DefaultPaymentExpDate**: null
- **DefaultPaymentTrackingNumber**: null
- **DefaultPaymentNameOnCard**: null
- **DefaultPaymentTypeID**: null
- **PricingPlanTypeID**: 10
- **PaymentTermsID**: 1
- **DiscountLevel**: 0
- **PricingLevel**: 1
- **PONumberRequired**: false
- **IndustryID**: null
- **OriginID**: null
- **Marketing3ID**: null
- **SalesPersonID1**: 10
- **SalesPersonID2**: null
- **SalesPersonID3**: null
- **TaxExemptExpDate**: null
- **CreditNumber**: null
- **PricingLevelID**: null
- **PromotionID**: null
- **UseTaxLookup**: false
- **HasServiceContract**: false
- **ServiceContractStartDate**: null
- **ServiceContractExpDate**: null
- **ServiceContractTypeID**: null
- **ServiceContractNotes**: null
- **DivisionID**: 10
- **RegionID**: null
- **PONumber**: null
- **PrimaryNumber**: (920) 743- 3353
- **PriNumberTypeID**: 10
- **PriNumberTypeText**: Office Phone
- **SecondaryNumber**: (920) 743- 3353
- **SecNumberTypeID**: 11
- **SecNumberTypeText**: Office Fax
- **IsClient**: true
- **IsVendor**: false
- **IsPersonal**: false
- **Is1099Vendor**: false
- **VendorPaymentTermsID**: null
- **MyAccountNumber**: null
- **DefaultShipMethodID**: null
- **ThirdNumber**: null
- **ThirdNumberTypeID**: null
- **ThirdNumberTypeText**: null
- **IsFullyTaxExempt**: false
- **VendorCreditBalance**: null
- **StageID**: 106
- **StageClassTypeID**: 5512
- **StageActivityID**: null
- **StageActivityClassTypeID**: null
- **LicenseKey**: null
- **DefaultPaymentMethodID**: null
- **PriContactNumber**: null
- **EntityUseCode**: null
- **EntityUseCodeName**: null
- **BillingFormattedNumber**: null
- **BillingContactNumber**: null
- **PrimaryContactNumber**: null

<!-- Verified against internal consistency checks 2026-02-09 -->
