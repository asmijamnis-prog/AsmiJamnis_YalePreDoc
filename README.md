# AsmiJamnis_YalePreDoc
Coding Sample
/*==============================================================================
   Project:  Geographic Population & Enrollment Analysis
   File:     Geo_Population_Regress.do
   Author:   Asmi Jamnis
   Date:     April 2026

   Description:
     Constructs catchment-zone population measures by combining home county
     and adjacent county ACS population data, merges with IPEDS enrollment,
     and estimates two-way fixed-effects regressions of log enrollment on
     log catchment population — overall and interacted with geographical regions

*==============================================================================
* STARTING DATASET: Population_enroll_IPEDS_05_24.dta
*==============================================================================

Units of Observation: unitid x neighbor county (one row per adjacent county pairing for each institution)

Key Variables already present
	unitid  			- IPEDS institution Identified
	instnm  			- Institution Name
	homecounty			- Name of Adjacent County
	homecountycode 		- FIPS code of the county the institution sits in
	neighborcountycode 	- FIPS code of one adjacent county	
	neighborname 		- Name of Adjacent County
	
		Adjaceny Structure: 
			The home-to-neighbor county pairs were pre-built using the US Census County Adjacency File.

NOT YET IN DATASET
	- ACS Population Counts (merged in Section 1)
	- IPEDS Enrollment & locale (merged in Section 4)

*/

*==============================================================================
* 0. SETTINGS & FILE PATHS
*==============================================================================

clear


* --- Working directory ---
cd "/Users/asmijamnis/UGRA Purdue Economics/Thesis/Ipeds Data"

* --- Input data paths ---
local pop_data    "Downloads/Total_Population_nhgis0009_csv/Population_2005_2024.dta"
local enroll_data "Documents/UGRA Purdue Economics/Thesis/Ipeds Data/IPEDS Enroll Data by Regional_Flagship.dta"
local locale_data "Downloads/LOCALE_Regional.dta"



*==============================================================================
* 1. BUILD DATASET — MERGE POPULATION (HOME COUNTY & NEIGHBOR COUNTY)
*==============================================================================

* --- 1a. Merge home county population ---
rename homecountycode countycode

merge m:1 countycode using "`pop_data'", ///
    keepusing(popu_0509 popu_1014 popu_1519 popu_2024)
keep if _merge == 3          // N = 420 obs
drop _merge

rename (popu_0509 popu_1014 popu_1519 popu_2024) ///
       (homepopu_0509 homepopu_1014 homepopu_1519 homepopu_2024)   // <-- rename to identify Home Couny Population

rename countycode homecountycode

* --- 1b. Merge neighbor county population ---
rename neighborcountycode countycode

merge m:1 countycode using "`pop_data'", ///
    keepusing(popu_0509 popu_1014 popu_1519 popu_2024)
keep if _merge == 3          // N = 420 obs
drop _merge

rename (popu_0509 popu_1014 popu_1519 popu_2024) ///
       (adjpopu_0509 adjpopu_1014 adjpopu_1519 adjpopu_2024)       // <-- rename to identify Adjacent County Population

rename countycode neighborcountycode


*==============================================================================
* 2. CONSTRUCT CATCHMENT ZONE POPULATION
* Catchment = Home County Population + Adjacent County Population
*==============================================================================

* Loop over ACS periods to build catchment = home county + all adjacent counties
foreach period in 0509 1014 1519 2024 {

    * Sum adjacent county populations within each university
    bysort unitid: egen total_adj_`period' = total(adjpopu_`period')

    * Catchment = home county pop + total adjacent county pop
    gen catchment_`period' = homepopu_`period' + total_adj_`period'

    label var catchment_`period' "Catchment Population (Home + Adjacent), `period'"
}

* Collapse to one row per university (keep catchment vars and identifiers)
collapse (first) instnm homecounty neighborname catchment_*, by(unitid)


*==============================================================================
* 3. RESHAPE & EXPAND TO UNIVERSITY-YEAR PANEL
*==============================================================================

* --- 3a. Reshape catchment wide → long (one row per unitid × ACS period) ---
reshape long catchment_, i(unitid) j(period) string

collapse (mean) catchment_, by(unitid period)

* --- 3b. Expand each 5-year ACS period into individual calendar years ---
expand 5

bysort unitid period: gen n = _n

gen year = .
replace year = 2005 + n - 1 if period == "0509"
replace year = 2010 + n - 1 if period == "1014"
replace year = 2015 + n - 1 if period == "1519"
replace year = 2020 + n - 1 if period == "2024"

drop n

* Verify panel is balanced
tab period
isid unitid year


*==============================================================================
* 4. MERGE ENROLLMENT & LOCALE
*==============================================================================

* --- 4a. Enrollment (IPEDS) ---
merge 1:1 unitid year using "`enroll_data'", keepusing(eftotlt)
drop if _merge != 3
drop _merge

* --- 4b. Locale classification ---
merge m:1 unitid using "`locale_data'", keepusing(locale)
drop if _merge != 3
drop _merge

* Collapse NCES locale codes into four groups
gen locale_group = .
replace locale_group = 1 if inlist(locale, 11, 12, 13)   // City
replace locale_group = 2 if inlist(locale, 21, 22, 23)   // Suburb
replace locale_group = 3 if inlist(locale, 31, 32, 33)   // Town
replace locale_group = 4 if inlist(locale, 41, 42, 43)   // Rural

label define loc_lbl 1 "City" 2 "Suburb" 3 "Town" 4 "Rural"
label values locale_group loc_lbl


*==============================================================================
* 5. GENERATE REGRESSION VARIABLES
*==============================================================================

gen ln_enroll = ln(eftotlt)
gen ln_popu   = ln(catchment_)

label var ln_enroll "Log Total Enrollment (IPEDS)"
label var ln_popu   "Log Catchment Zone Population"


*==============================================================================
* 6. REGRESSIONS
*==============================================================================

* --- 6a. Baseline: university & year fixed effects only ---
reghdfe ln_enroll ln_popu, absorb(unitid year)

* --- 6b. Main model: population × locale interaction ---
reghdfe ln_enroll c.ln_popu##i.locale_group, absorb(unitid year)



*==============================================================================
* END OF FILE
*==============================================================================
