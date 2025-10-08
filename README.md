# 🏡 Nashville Housing Data Cleaning in SQL Server Management Studio

📘 **Overview**<br>
This project demonstrates a complete data cleaning process using Microsoft SQL Server Management Studio (SSMS). The goal of this project is to transform raw housing data into a clean format by standardizing columns, handling nulls, splitting values, removing duplicates, and dropping unnecessary data.

Nashville Housing Data is a Real Estate dataset that contains property records including sale dates, prices, owner information, and addresses.

***

🧰 **Tools Used**
- SQL Server Management Studio (SSMS 21)
- SQL Server Express (Local Instance)
- Dataset: [NashvilleHousing.csv](https://github.com/larsyarafina/NashvilleHousing/blob/main/NashvilleHousing.xlsx)

***

🗂️ Dataset Description
| **Column Name**   | **Description**                                                                   |
| ----------------- | --------------------------------------------------------------------------------- |
| `UniqueID`        | Unique identifier for each property record                                        |
| `ParcelID`        | Unique parcel number identifying each land lot                                    |
| `LandUse`         | Classification of how the land is being used (e.g., Single Family, Duplex, Condo) |
| `PropertyAddress` | Full address of the property (street, city, state)                                |
| `SaleDate`        | Original date of the property sale                                                |
| `SalePrice`       | Final selling price of the property                                               |
| `LegalReference`  | Reference to the legal transaction document                                       |
| `SoldAsVacant`    | Indicates if the property was vacant at the time of sale (`Yes` / `No`)           |
| `OwnerName`       | Full name of the property owner(s)                                                |
| `OwnerAddress`    | Mailing address of the property owner                                             |
| `Acreage`         | Size of the property in acres                                                     |
| `TaxDistrict`     | Tax district where the property is located                                        |
| `LandValue`       | Assessed land value by the local tax authority                                    |
| `BuildingValue`   | Assessed building value (structure only)                                          |
| `TotalValue`      | Combined assessed value of land and buildings                                     |
| `YearBuilt`       | Year the property structure was originally built                                  |
| `Bedrooms`        | Number of bedrooms in the property                                                |
| `FullBath`        | Number of full bathrooms                                                          |
| `HalfBath`        | Number of half bathrooms                                                          |

***

🧹 **Data Cleaning Steps**
1. Standardize Date Format\
Converted inconsistent date formats into a proper `DATE` type for better querying and analysis.
```sql
ALTER TABLE PortfolioProjects.dbo.NashvilleHousing
ADD SaleDateConverted DATE;

UPDATE PortfolioProjects.dbo.NashvilleHousing
SET SaleDateConverted = CONVERT(DATE, SaleDate);
```

2. Populate Missing Property Address\
Filed missing `PropertyAddress` values using matching `ParcelID` records.
```sql
select a.ParcelID, b.ParcelID, a.PropertyAddress, b.PropertyAddress, isnull(a.PropertyAddress, b.PropertyAddress)
from PortfolioProjects.dbo.NashvilleHousing a
JOIN PortfolioProjects.dbo.NashvilleHousing b
	on a.ParcelID = b.ParcelID
	and a.[UniqueID ] <> b.[UniqueID ]
where a.PropertyAddress is null

update a
set PropertyAddress = isnull(a.PropertyAddress, b.PropertyAddress)
from PortfolioProjects.dbo.NashvilleHousing a
JOIN PortfolioProjects.dbo.NashvilleHousing b
	on a.ParcelID = b.ParcelID
	and a.[UniqueID ] <> b.[UniqueID ]
where a.PropertyAddress is null
```

3. Split Address Fields into Separate Columns
Separated combined address fields into `Address`, `City`, and `State` for both property and owner.

For `PropertyAdress`
```sql
select
substring(PropertyAddress, 1, charindex(',', PropertyAddress) -1) as Address,
substring(PropertyAddress, charindex(',', PropertyAddress) +1, len(PropertyAddress)) as Address
from PortfolioProjects.dbo.NashvilleHousing;

ALTER TABLE PortfolioProjects.dbo.NashvilleHousing
ADD PropertySplitAddress NVARCHAR(255),
    PropertySplitCity NVARCHAR(255);

UPDATE PortfolioProjects.dbo.NashvilleHousing
SET PropertySplitAddress = SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) - 1),
    PropertySplitCity = LTRIM(SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) + 1, LEN(PropertyAddress)));
```
For `PropertyOwner`
```sql
SELECT 
PARSENAME(REPLACE(OwnerAddress,',','.'),3),
PARSENAME(REPLACE(OwnerAddress,',','.'),2),
PARSENAME(REPLACE(OwnerAddress,',','.'),1)
FROM PortfolioProjects.dbo.NashvilleHousing;

ALTER TABLE PortfolioProjects.dbo.NashvilleHousing
ADD OwnerSplitAddress NVARCHAR(255),
    OwnerSplitCity NVARCHAR(255),
	OwnerSplitState NVARCHAR(255);

UPDATE PortfolioProjects.dbo.NashvilleHousing
SET OwnerSplitAddress = PARSENAME(REPLACE(OwnerAddress,',','.'),3),
    OwnerSplitCity = PARSENAME(REPLACE(OwnerAddress,',','.'),2),
	OwnerSplitState = PARSENAME(REPLACE(OwnerAddress,',','.'),1);
```

4. Standardize Categorical Values
Replaced inconsistent labels in `SoldAsVacant` from `'Y'`/`'N' to `'Yes'`/`'No'`
```sql
select SoldAsVacant,
	case when SoldAsVacant ='Y' then 'Yes'
		 when SoldAsVacant ='N' then 'No'
		 else SoldAsVacant
		 end
FROM PortfolioProjects.dbo.NashvilleHousing;

UPDATE PortfolioProjects.dbo.NashvilleHousing
SET SoldAsVacant = case when SoldAsVacant ='Y' then 'Yes'
		 when SoldAsVacant ='N' then 'No'
		 else SoldAsVacant
		 end;
```

5. Remove duplicates
Used a CTE (`Row_Number()`) to identify and delete duplicate rows based on key property attributes.
```sql
WITH RowNumCTE AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY ParcelID,
                         PropertyAddress,
                         SalePrice,
                         SaleDate,
                         LegalReference
            ORDER BY UniqueID
        ) AS row_num
    FROM PortfolioProjects.dbo.NashvilleHousing
)
DELETE FROM RowNumCTE
WHERE row_num > 1;
```

6. Drop Unused Columns
Removed redundant or unneeded columns after data normalization
```sql
alter table PortfolioProjects.dbo.NashvilleHousing
drop column OwnerAddress, TaxDistrict, PropertyAddress, SaleDate;
```
