# IEO-cognitive-functioning
## Code by Anja Leist for doi: 10.1016/j.ssmph.2021.100837
Please cite this paper when using the code:
Leist, A. K., Bar-Haim, E., & Chauvel, L. (2021). Inequality of educational opportunity at time of schooling predicts cognitive functioning in later adulthood. SSM - Population Health, doi: 10.1016/j.ssmph.2021.100837.
## Data availability
This paper uses data from the Survey of Health, Ageing and Retirement in Europe (SHARE) 2004–2017; the World Bank World Development Indicators and World Bank Global Database on Intergenerational Mobility; the UN Human Development Reports; and the WHO Global Health Observatory. The responsibility for all conclusions drawn from the data lies entirely with the authors. Specifically, this paper uses data from SHARE Waves 1, 2, 3, 4, 5, 6, and 7 (DOIs: 10.6103/SHARE.w1.700, 10.6103/SHARE.w2.700, 10.6103/SHARE.w3.700, 10.6103/SHARE.w4.700, 10.6103/SHARE.w5.700, 10.6103/SHARE.w6.700, 10.6103/SHARE.w7.700), see Börsch-Supan et al. (2013) for methodological details. This paper additionally uses parts of Stata code published with the easySHARE data set (DOI:  10.6103/SHARE.easy.700), see Gruber et al. (2019) for methodological details. The SHARE data collection has been funded by the European Commission through FP5 (QLK6-CT-2001-00360), FP6 (SHARE-I3: RII-CT-2006-062,193, COMPARE: CIT5-CT-2005-028,857, SHARELIFE: CIT4-CT-2006-028,812), FP7 (SHARE-PREP: GA N◦211,909, SHARE-LEAP:GA N◦227,822, SHARE M4: GA N◦261,982), and Horizon 2020 (SHARE-DEV3: GA N◦676,536, SERISS: GA N◦654,221) and by DG Employment, Social Affairs & Inclusion. Additional funding from the German Ministry of Education and Research, the Max Planck Society for the Advancement of Science, the U.S. National Institute on Aging (U01_AG09740-13S2, P01_AG005842, P01_AG08291, P30_AG12815, R21_AG025169, Y1-AG-4553-01, IAG_BSR06-11, OGHA_04–064, HHSN271201300071C), and from various national funding sources is gratefully acknowledged (see www.share-project.org).
## Funding
This work was supported by the European Research Council (grant agreement no. 803239, 2019–2023, to AKL); and the National Research Fund Luxembourg (FNR/P11/05 and FNR/P11/05 bis, 2012–2018, to LC).
## Code
The dataset is based on the easySHARE dataset that is a bit pre-prepared by the SHARE team, plus some additional variables that have been added to the easySHARE dataset (e.g., fluency, childhood variabless, isco information, deceased at follow-up).
Since the cognitive scores need to be standardized several times for the analyses that are either separate by gender or for the pooled sample, in the end one should have several datasets: "both" at the end means both sexes, and there should be also datasets for men and women separately, which will be read into R.

```
clear all
global pathdata "define your global paths here"
global pathout " define your global paths here"
use $pathdata\easySHARE_rel7-0-0.dta, clear
describe
sort mergeid
bysort mergeid: gen nrobs = 1 if mergeid!=mergeid[_n-1]
su nrobs
su nrobs if age>49.999
ta ch001_
ta deceased
ta female if deceased==1
```
## Cohorts
Define eligibility by cohort first and then define "running", the other way round is also fine, first drop younger respondents.

```
by mergeid: gen ineligible = (age<50)
ta ineligible
by mergeid: egen mineligible = max(ineligible)
ta mineligible
drop if mineligible==1
ta nrobs
drop ineligible mineligible
```
Age is provided by the SHARE team who did plausibility checks; missing birth months were recoded to June, regenerate birthyear with those clean data to create indicator for cohort; reduce errors due to rounding: assign born in 1939.5=1940s cohort.
```
su int_year
ta int_month
gen birthyear = ((int_year*12 + int_month) - (age*12))/12
su birthyear
su birthyear if birthyear >1939.4 & birthyear  < 1970
```
The GDIM database has ten-year cohorts (https://www.worldbank.org/en/topic/poverty/brief/what-is-the-global-database-on-intergenerational-mobility-gdim), three of the cohorts fit in the age range of SHARE.
Include all who were eligible in terms of cohort membership at start and became eligible at a follow-up wave: done already by only including 50+ year-olds.
```
recode birthyear(1939.5/1949.9999=1)(1950/1959.9999=2)(1960/1963.9999=3)(else=.), gen(cohor_)
ta cohor_
by mergeid: egen cohort = min(cohor_)
label define lcohort 1 "1940-1949" 2 "1950-1959" 3 "1960-1969"
label values cohort lcohort
ta cohort
su nrobs if cohort==.
drop if cohort==.
ta nrobs
ta nrobs if country==31
```
## Getting around the study design complexities
Drop all participants with less than 2 interviews.
```
su nrobs if wavepart<12 & age>49.999
su nrobs if wavepart==3 & age>49.999
su nrobs if wavepart==7 & age>49.999
drop if wavepart<12
gen wavepartstring = string(wavepart)
su nrobs if length(wavepartstring)==2 & strpos(wavepartstring, "3")
su nrobs if length(wavepartstring)==2 & strpos(wavepartstring, "7")
```
Drop all participants with two measurements of which one was SHARELIFE.
```
drop if length(wavepartstring)==2 & strpos(wavepartstring, "3")
drop if length(wavepartstring)==2 & strpos(wavepartstring, "7")
ta nrobs 
```
Identify number of any cognitive test per assessment.
```
recode recall_* fluency (min/-1=.)
bysort mergeid: gen outcome = 1 if (recall_1!=. | recall_2!=. | fluency!=.)
ta outcome
```
Creating counter for number of cognitive assessments: "running".
```
by mergeid: gen running = sum(outcome)
drop if outcome==.
ta running
drop if running==0
ta running outcome 
ta running 
list mergeid running outcome nrobs cohort in 1/100, noobs sepby(mergeid)
```
Total number of observations:
```
bysort mergeid: egen totalobs=count(running)
ta totalobs if mergeid!=mergeid[_n-1] & age>49.999 & totalobs>1
su totalobs if running==1
sort mergeid
ta outcome if mergeid!=mergeid[_n-1] 
ta country if mergeid!=mergeid[_n-1] 
ta country cohort if mergeid!=mergeid[_n-1] 
```
Select participants with two cognitive assessments:
```
drop if totalobs<2
ta running
ta nrobs
```
Select participating SHARE countries that are also in GDIM.
```
drop if country==31 // drop Luxembourg as it is not in GDIM
ta nrobs //final number of respondents before selecting according to individual covariates
```



## Individual-level covariates
Number of children:
```
ta ch001_, nola
recode ch001_ (min/-1=.)
ta ch001_
su ch001_
by mergeid: gen children = ch001_[1]
ta children
su children
gen catchildren=children
recode catchildren (3/max=3)
ta catchildren
```
Education:
```
recode isced1997_r (min/-1=.)
ta isced1997_r
recode isced1997_r (0/2=1)(3=2)(4/6=3)(else=.), gen(edu_)
ta edu_
by mergeid: gen edu = edu_[1]
label define ledu 1 "ISCED 0-2" 2 "ISCED 3" 3 "ISCED 4-6"
label values edu ledu
ta edu
```
Cohabitation/living-with-partner status:
```
ta mar_stat
bysort mergeid: gen living = mar_stat[1]
recode living (min/-1=.)(1/3=1)(4/6=0)
label define lliving 1 "married/living with partner" 0 "living alone"
label values living lliving
ta living if mergeid!=mergeid[_n-1] 
```
Current job situation:
```
ta ep005_
by mergeid: gen currentsit = ep005_[1]
recode currentsit (min/-1=.)(97=.)
label define lcurrentsit 1 "retired" ///
2 "employed or self-employed" ///
3 "unemployed" ///
4 "permanently sick or disabled" ///
5 "homemaker"
label values currentsit lcurrentsit
ta currentsit
```
Childhood variables:
```
recode books_age10 maths_age10 language_age10 (min/-1=.)
ta books_age10 //number of books in household at age 10
codebook books_age10
by mergeid: egen books = max(books_age10)
label define lbooks 1 "1. none or very few (0-10 books)" ///
					2 "2. enough to fill one shelf (11-25 books)" ///
					3 "3. enough to fill one bookcase (26-100 books)" ///
					4 "4. enough to fill two bookcases (101-200 books)" ///
					5 "5. enough to fill two or more bookcases (more than 200)"
label values books lbooks
ta books
ta maths_age10 //skills in mathematics relative to peers at age 10
by mergeid: egen maths10 = max(maths_age10)
label define lskills 1 "1. much better" ///
					2 "2. better" ///
					3 "3. about the same" ///
					4 "4. worse" ///
					5 "5. much worse"
label values maths10 lskills
ta maths10 
ta language_age10 //skills in language relative to peers at age 10
by mergeid: egen language10 = max(language_age10)
label values language10 lskills
ta language10
ta childhood_health //health at age 10
recode childhood_health (min/-1=.)(6=.) //health varied a great deal: N=274
ta childhood_health
by mergeid: egen childhealth = max(childhood_health)
label define lhealth 1 "1. Excellent" ///
					2 "2. Very good" ///
					3 "3. Good" ///
					4 "4. Fair" ///
					5 "5. Poor"
label values childhealth lhealth
ta childhealth
```
Adulthood variables:
```
ta sphus //self-rated health at age 10
recode sphus (min/-1=.)
by mergeid: gen srh = sphus[1]
recode srh (1/4=0)(5=1)
label define lpoor 1 "Poor" ///
					0 "Fair or better"
label values  srh lpoor
ta srh if running==1
ta chronic_mod //number of chronic conditions at first interview
recode chronic_mod (min/-1=.)
by mergeid: gen chronic = chronic_mod[1]
ta chronic
ta eurod // number of depressive symptoms at first interview
recode eurod (min/-1=.)
by mergeid: gen depressed = eurod[1]
ta depressed
```
Occupational level (ISCO) of current or last job: added to easySHARE; wave 1 as base information, if missing=current job of further waves, if missing=last job of further waves.
```
ta jisco // from wave 1
ta cisco //current job from further waves
ta lisco //last job from further waves
recode jisco (0=10)
recode jisco cisco lisco (min/-1=.)
ta jisco
ta cisco
ta lisco
by mergeid: gen isco=jisco
by mergeid: replace isco=cisco if isco==.
by mergeid: replace isco=lisco if isco==.
ta isco
lab var isco "ISCO occupation current/last job"
lab define lisco 1 "Senior official, manager" ///
			2 "Professional" ///
			3 "Technician" ///
			4 "Clerk" ///
			5 "Services, sales worker" ///
			6 "Skilled agricultural, fishery worker" ///
			7 "Craft, trades worker" ///
			8 "Plant, machine operator, assembler" ///
			9 "Elementary occupation" ///
			10 "Armed forces"
label values isco lisco
ta isco
```
Exclude those from dataset who were not schooled in the country of interview, "heresince" added in easySHARE as date of coming to current country of residence; plus "schooled abroad" as indicator for born outside country of residence + came to country of residence after age 10 (as we suggest after this age most of the sorting, tracking, and labeling takes place.
```
su heresince
ta heresince
recode heresince (min/-1=.)
su dn003_mod
gen age_arrival=heresince-dn003_mod
su age_arrival
gen schooledabroad=(age_arrival < 10)
ta schooledabroad
ta country schooledabroad if running==1
```
Select according to eligibility criteria: info on ISCED level, info on current job sit, and living in the country of residence since age 10.
```
ta edu if running==1
ta living if running==1
ta currentsit if running==1
ta schooledabroad if running==1
bysort mergeid: gen miss_edu=1 if edu[1]==.
bysort mergeid: gen miss_living=1 if living[1]==.
bysort mergeid: gen miss_currentsit=1 if currentsit[1]==.
bysort mergeid: gen max_schooledabroad=1 if schooledabroad[1]==1
drop if miss_edu==1
drop if miss_living==1
drop if miss_currentsit==1
drop if max_schooledabroad==1
```
Description of sample size.
```
ta running 
bysort female: ta running
ta running if srh!=. & chronic!=. & depressed!=.
bysort female: ta running if srh!=. & chronic!=. & depressed!=.
ta running if srh!=. & chronic!=. & depressed!=. & books!=. & maths10!=. & language10!=. & childhealth!=.
bysort female: ta running if srh!=. & chronic!=. & depressed!=. & books!=. & maths10!=. & language10!=. & childhealth!=.
bysort edu: tabstat recall_1, by(running)
bysort edu: tabstat recall_2, by(running)
bysort edu: tabstat fluency, by(running)
```

## Macro-level determinants
Create country-cohort indicators for country of schooling and residence and birth cohort `cohcoun` and `id`.
```
ta country
ta country, nola
gen cohcoun=100*cohort+country
ta cohcoun
label define lcohcoun 	111 "Austria 1940-49" ///
	112 "Germany 1940-49" ///
	113 "Sweden 1940-49" ///
	114 "Netherlands 1940-49" ///
	115 "Spain 1940-49" ///
	116 "Italy 1940-49"  ///
	117 "France 1940-49"  ///
	118 "Denmark 1940-49" ///
	119 "Greece 1940-49" ///
	120 "Switzerland 1940-49" ///
	123 "Belgium 1940-49" ///
	125 "Israel 1940-49" ///
	128 "Czech Republic 1940-49" ///
	129 "Poland 1940-49" ///
	131 "Luxembourg 1940-49" ///
	133 "Portugal 1940-49" ///
	134 "Slovenia 1940-49" ///
	135 "Estonia 1940-49" ///
	211 "Austria 1950-59" ///
	212 "Germany 1950-59" ///
	213 "Sweden 1950-59" ///
	214 "Netherlands 1950-59" ///
	215 "Spain 1950-59" ///
	216 "Italy 1950-59" ///
	217 "France 1950-59" ///
	218 "Denmark 1950-59" ///
	219 "Greece 1950-59" ///
	220 "Switzerland 1950-59" ///
	223 "Belgium 1950-59" ///
	225 "Israel 1950-59" ///
	228 "Czech Republic 1950-59" ///
	229 "Poland 1950-59" ///
	231 "Luxembourg 1950-59" ///
	233 "Portugal 1950-59" ///
	234 "Slovenia 1950-59" ///
	235 "Estonia 1950-59" ///
	311 "Austria 1960-69" ///
	312 "Germany 1960-69" ///
	313 "Sweden 1960-69" ///
	314 "Netherlands 1960-69" ///
	315 "Spain 1960-69" ///
	316 "Italy 1960-69" ///
	317 "France 1960-69" ///
	318 "Denmark 1960-69" ///
	319 "Greece 1960-69" ///
	320 "Switzerland 1960-69" ///
	323 "Belgium 1960-69" ///
	325 "Israel 1960-69" ///
	328 "Czech Republic 1960-69" ///
	329 "Poland 1960-69" ///
	331 "Luxembourg 1960-69" ///
	333 "Portugal 1960-69" ///
	334 "Slovenia 1960-69" ///
	335 "Estonia 1960-69" ///
	label values cohcoun lcohcoun
ta cohcoun if cohort!=. & mergeid!=mergeid[_n-1]
ta country cohort if mergeid!=mergeid[_n-1] 
```
Second indicator country `id` based on `cohcoun` for merging.
```
decode cohcoun,gen(cntco)
rename cohort ocohort
gen cohort=substr(cntco,-2,2)
destring cohort, force replace
replace cohort=cohort-9
rename country ocountry
gen country=word(cntco,1)
replace country="Czech Republic" if country=="Czech"
gen id=country+substr(cntco,-5,2)
ta id
save $pathout\easyCOGN, replace
```
### Inequality of Educational Opportunity (IEO)
From World Bank dataset; Luxembourg data not available; preparation of ieo_long.dta by Eyal Bar-Haim, 18 July 2019.
```
use $pathout\easyCOGN, clear
drop _merge
merge m:1 id using $pathdata\ieo_long.dta, force
rename cor ieo
lab var ieo "IEO by WorldBank"
ta country if mergeid!=mergeid[_n-1] 
su ieo if mergeid!=mergeid[_n-1] 
tab country, su(ieo)
tab country cohort, su(ieo)
keep if _merge==3 //select only individuals with IEO information
drop _merge
sort mergeid
save $pathout\easyCOGN_macro, replace
```
### Gross Domestic Income per Capita PPP (GDP)
GDP taken from http://blogs.worldbank.org/opendata/introducing-online-guide-world-development-indicators-new-way-discover-data-development; gdp per capita PPP = dataset gdp2, already manually cut down in Excel; WDI dataset last updated 10 July 2019.
```
use $pathdata\gdp2, clear
describe
rename CountryName country
codebook D
rename D gdp
save $pathdata\gdppercapitappp, replace
use $pathout\easyCOGN_macro, clear
sort country
merge m:1 country using $pathdata\gdppercapitappp.dta, force
keep if _merge==3
drop _merge
tab country, su(gdp)
save $pathout\easyCOGN_macro, replace
```
### Human Development Index (HDI)
HDI taken from http://hdr.undp.org/en/data#, already manually cut down in Excel; data last updated 31 January 2019.
```
use $pathdata\HDI, clear
ta country
replace country="Czech Republic" if country=="Czechia"
describe
save $pathdata\HDI, replace
use $pathout\easyCOGN_macro, clear
sort country
merge m:1 country using $pathdata\HDI.dta, force
ta country if _merge==3
keep if _merge==3
drop _merge
tab country, su(HDI)
save $pathout\easyCOGN_macro, replace
```
### Healthy life expectancy at age 60 (HALE)
HDI taken from http://apps.who.int/gho/data/view.main.HALEXv?lang=en; 17 July 2019, already manually cut down in Excel.
```
use $pathdata\hle60bothsexes, clear
ta country
replace country="Czech Republic" if country=="Czechia"
save $pathdata\hle60bothsexes, replace
use $pathout\easyCOGN_macro, clear
ta country
sort country
merge m:1 country using $pathdata\hle60bothsexes.dta, force
ta country if _merge==3 
keep if _merge==3
drop _merge
tab country, su(hle60)
sort mergeid //to have unique individuals in the next line
ta country if mergeid!=mergeid[_n-1]

list mergeid running ieo gdp hle60 in 1/20, noobs sepby(mergeid)
save $pathout\easyCOGN_macro, replace
```
## Prepare cognitive test measures 
Standardize scores of the three cognitive tests: immediate recall 1 (imm), delayed recall 2 (del), fluency (flu); create average cognitive z-score (ctot) for graphs and interaction analyses
```
use $pathout\easyCOGN_macro, clear
recode recall* fluency (min/-1=.)
```
Indicator to drop those with all cognitive scores missing at a particular t.
```
by mergeid running, sort: gen miss = (missing(recall_1) & missing(recall_2) & missing(fluency))
ta miss
drop if miss==1
```
Explore:
```
summarize recall_1 recall_2 fluency
tabstat recall* fluency, statistics(mean) by(cohort)
tab cohort running, su(recall_1)
tab cohort running, su(recall_2)
tab cohort running, su(fluency)
preserve
keep if running==1
su recall_1 recall_2 fluency
restore
preserve
keep if cohort==40
tabstat recall_1 recall_2 fluency, by(running)
restore
preserve
keep if cohort==50
tabstat recall_1 recall_2 fluency, by(running)
restore
preserve
keep if cohort==60
tabstat recall_1 recall_2 fluency, by(running)
restore
preserve
keep if running==1
tab cohcoun, su(age)
restore
```
Reduce variables and reshape, save to new dataset and merge later.
```
keep mergeid running cohort recall_* fluency edu
reshape wide recall_1 recall_2 fluency edu, i(mergeid) j(running)
save $pathout\easyreduced, replace
```
## Prepare cognitive scores: Both sexes
All cognitive scores are standardized (zimm, zdel, zflu) with wave 1 mean and std for each cohort.
```
forvalues i = 1/6 {
gen zimm`i'=.
gen zdel`i'=.
gen zflu`i'=.
}
su z*

*pretty sure this could be done more efficiently
levelsof cohort, local(cohorts)
foreach c of local cohorts {
su recall_11 if cohort == `c' 
gen immmean`c' = r(mean)
gen immsd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zimm`i' = (recall_1`i' - immmean`c')/immsd`c'  if cohort == `c' 
}
}
foreach c of local cohorts {
su recall_21 if cohort == `c' 
gen delmean`c' = r(mean)
gen delsd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zdel`i' = (recall_2`i' - delmean`c')/delsd`c'  if cohort == `c' 
}
}
foreach c of local cohorts {
su fluency1 if cohort == `c' 
gen flumean`c' = r(mean)
gen flusd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zflu`i' = (fluency`i' - flumean`c')/flusd`c'  if cohort == `c' 
}
}
tabstat z*, statistics(mean) by(cohort)
tabstat z*, statistics(n) by(cohort)
forvalues i = 1/6 {
local cogn "zimm`i' zdel`i' zflu`i'"
egen ctot`i' = rmean(`cogn')
}
tabstat ctot*, statistics(mean) by(cohort)
tabstat ctot*, statistics(n) by(cohort)
keep mergeid z* c*
sort mergeid
reshape long zimm zdel zflu ctot, i(mergeid) j(running)
sort mergeid running
save $pathout\easyreduced, replace
 use $pathdata\easyCOGN_macro.dta, clear
sort mergeid running
merge 1:1 mergeid running using $pathdata\easyreduced, assert(1 2 3)
describe
keep if _merge==3
save $pathout\easyCOGN_macro_both, replace
```

## Prepare cognitive scores: Women
```
use $pathout\easyCOGN_macro, clear
keep if female==1
recode recall* fluency (min/-1=.)
```
Only those cognitive tests for those who do not have neither test at a particular t.
```
by mergeid running, sort: gen missw = (missing(recall_1) & missing(recall_2) 
& missing(fluency))
ta missw
drop if missw==1
```
Reduce variables and reshape, save to new dataset and merge later.
```
keep mergeid running cohort recall_* fluency edu
*reshape wide to have fewer problems with the loops
reshape wide recall_1 recall_2 fluency edu, i(mergeid) j(running)
save $pathout\easywomen, replace
```
All cognitive scores standardized with wave 1 mean and std for each cohort.
```
forvalues i = 1/6 {
gen zimm`i'=.
gen zdel`i'=.
gen zflu`i'=.
}
levelsof cohort, local(cohorts)
foreach c of local cohorts {
su recall_11 if cohort == `c' 
gen immmean`c' = r(mean)
gen immsd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zimm`i' = (recall_1`i' - immmean`c')/immsd`c'  if cohort == `c' 
}
}
foreach c of local cohorts {
su recall_21 if cohort == `c' 
gen delmean`c' = r(mean)
gen delsd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zdel`i' = (recall_2`i' - delmean`c')/delsd`c'  if cohort == `c' 
}
}
foreach c of local cohorts {
su fluency1 if cohort == `c' 
gen flumean`c' = r(mean)
gen flusd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zflu`i' = (fluency`i' - flumean`c')/flusd`c'  if cohort == `c' 
}
}
tabstat z*, statistics(mean) by(cohort)
tabstat z*, statistics(n) by(cohort)
forvalues i = 1/6 {
local cogn "zimm`i' zdel`i' zflu`i'"
egen ctot`i' = rmean(`cogn')
}
keep mergeid z* c*
sort mergeid
reshape long zimm zdel zflu ctot, i(mergeid) j(running)
sort mergeid running
save $pathout\easywomen, replace
 use $pathdata\easyCOGN_macro.dta, clear
sort mergeid running
merge 1:1 mergeid running using $pathdata\easywomen, assert(1 2 3)
describe
keep if _merge==3
keep mergeid running age female cohort country* cohcoun edu* isco ///
	children catchildren living currentsit z* ctot ieo gdp hle HDI books maths10 language10 childhealth ///
	srh chronic depressed schooledabroad deceased
	gen cage = (age-50)/10
lab var cage "age centered at 50 in decades"
by mergeid, sort: gen practice = (running==1)
lab var practice "indicator of first testing"
gen cagesquared = cage*cage
```
Center macrolevel determinants:
```
su ieo
gen cieo = ieo - r(mean)
su gdp
gen cgdp = gdp - r(mean)
su hle
gen chle = hle60 - r(mean)
su HDI
gen chdi = HDI - r(mean)
save $pathout\easyCOGN_mixedwomen, replace
```
## Prepare cognitive scores: Men
```
use $pathout\easyCOGN_macro, clear
keep if female==0
recode recall* fluency (min/-1=.)
```
Only those cognitive tests for those who do not have neither test at a particular t.
```
by mergeid running, sort: gen missm = (missing(recall_1) & missing(recall_2) 
& missing(fluency))
ta missm
drop if missm==1
```
Reduce variables and reshape, save to new dataset and merge later.
```
keep mergeid running cohort recall_* fluency edu
*reshape wide to have fewer problems with the loops
reshape wide recall_1 recall_2 fluency edu, i(mergeid) j(running)
save $pathout\easymen, replace
```
All cognitive scores standardized with wave 1 mean and std for each cohort.
```
forvalues i = 1/6 {
gen zimm`i'=.
gen zdel`i'=.
gen zflu`i'=.
}
*pretty sure this could be done more efficiently
levelsof cohort, local(cohorts)
foreach c of local cohorts {
su recall_11 if cohort == `c' 
gen immmean`c' = r(mean)
gen immsd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zimm`i' = (recall_1`i' - immmean`c')/immsd`c'  if cohort == `c' 
}
}
foreach c of local cohorts {
su recall_21 if cohort == `c' 
gen delmean`c' = r(mean)
gen delsd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zdel`i' = (recall_2`i' - delmean`c')/delsd`c'  if cohort == `c' 
}
}
foreach c of local cohorts {
su fluency1 if cohort == `c' 
gen flumean`c' = r(mean)
gen flusd`c' = r(sd)
}
foreach c of local cohorts {
forvalues i = 1/6 {
replace zflu`i' = (fluency`i' - flumean`c')/flusd`c'  if cohort == `c' 
}
}
tabstat z*, statistics(mean) by(cohort)
tabstat z*, statistics(n) by(cohort)
forvalues i = 1/6 {
local cogn "zimm`i' zdel`i' zflu`i'"
egen ctot`i' = rmean(`cogn')
}
keep mergeid z* c*
sort mergeid
reshape long zimm zdel zflu ctot, i(mergeid) j(running)
sort mergeid running
save $pathout\easymen, replace
 use $pathdata\easyCOGN_macro.dta, clear
sort mergeid running
merge 1:1 mergeid running using $pathdata\easymen, assert(1 2 3)
describe
keep if _merge==3
keep mergeid running age female cohort country* cohcoun edu* isco ///
	children catchildren living currentsit z* ctot ieo gdp hle HDI books maths10 language10 childhealth ///
	srh chronic depressed schooledabroad deceased
gen cage = (age-50)/10
lab var cage "age centered at 50 in decades"
by mergeid, sort: gen practice = (running==1)
lab var practice "indicator of first testing"
gen cagesquared = cage*cage
```
Center macrolevel determinants:
```
su ieo
gen cieo = ieo - r(mean)
su gdp
gen cgdp = gdp - r(mean)
su hle
gen chle = hle - r(mean)
su HDI
gen chdi = HDI - r(mean)
save $pathout\easyCOGN_mixedmen, replace
```
## Both sexes
```
use $pathout\easyCOGN_macro_both, clear
keep mergeid running age female cohort country* cohcoun edu* isco ///
	children catchildren living currentsit z* ctot ieo gdp hle HDI books maths10 language10 childhealth ///
	srh chronic depressed schooledabroad deceased
gen cage = (age-50)/10
lab var cage "age centered at 50 in decades"
by mergeid, sort: gen practice = (running==1)
lab var practice "indicator of first testing"
gen cagesquared = cage*cage
```
Center macrolevel determinants:
```
su ieo
gen cieo = ieo - r(mean)
su gdp
gen cgdp = gdp - r(mean)
su hle
gen chle = hle - r(mean)
su HDI
gen chdi = HDI - r(mean)
save $pathout\easyCOGN_mixedboth, replace
by mergeid: egen last = max(age)
by mergeid: egen first = min(age)
by mergeid: gen length = last - first
*median and IQR
su length, d
use $pathdata\easyCOGN, clear
describe
bysort mergeid: gen firstrecall_1 = recall_1[1]
sort mergeid
by mergeid: gen firstimm = recall_1[1]
by mergeid: gen firstdel = recall_2[1]
by mergeid: gen firstflu = fluency[1]
su firstrecall_1
su firstdel
su firstflu
```

# R
Install all missing packages first with the command `install.packages`, e.g. `install.packages("sjstats")`.
```
library(haven)
library(tidyverse)
library(sjstats)
library(lme4)
library(lmerTest)
library(optimx)
library(sjPlot)
library(ggplot2)
library(broom)
library(table1)
library(ggrepel)
options(scipen = 999)
```
## Read in datasets for men, women and both sexes/genders
```
men <- read_dta("your global path/dataset.dta")
#use only men who were schooled in the country of residence
men<-filter(men,schooledabroad==0)
women <- read_dta("your global path/dataset.dta")
#use only women who were schooled in the country of residence, delete 1 outlier
women<- filter(women, ctot<8,schooledabroad==0)
both<- read_dta("your global path/dataset.dta")
both<-filter(both,ctot<8,schooledabroad==0)
```
Read in aggregate dataset for Graph 1:
```
diff2<- read_dta("your global path/dataset.dta")
```
## lmer tables
### Women
```
modelw1 <- lmer(zimm ~ cage + cagesquared + practice +
                    factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                    cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=women, REML = FALSE, 
                  control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelw1)
vcov(modelw1)

modelw2 <- lmer(zdel ~ cage + cagesquared  + practice + 
                   factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                   cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=women, REML = FALSE, 
                 control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelw2)

modelw3 <- lmer(zflu ~ cage + cagesquared  + practice + 
                  factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                  cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=women, REML = FALSE, 
                control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelw3)
```
### Men
```
modelm1 <- lmer(zimm ~ cage + cagesquared  + practice + 
                  factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                  cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=men, REML = FALSE, 
                control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelm1)

modelm2 <- lmer(zdel ~ cage + cagesquared  + practice + 
                  factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                  cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=men, REML = FALSE, 
                control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelm2)

modelm3 <- lmer(zflu ~ cage + cagesquared  + practice + 
                  factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                  cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=men, REML = FALSE, 
                control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelm3)

tab_model(modelw1,modelw2,modelw3,modelm1,modelm2,modelm3, show.est=FALSE, show.std="std", collapse.ci=TRUE, p.style =c("asterisk"), file = "your global path/Table2.doc")
```

### Women, controlling for hdi
```
modelw1h <- lmer(zimm ~ cage + cagesquared  + practice +
                   factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed +
                   chdi + cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=women, REML = FALSE, 
                 control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelw1h)

modelw2h <- lmer(zdel ~ cage + cagesquared   + practice +
                    factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                    chdi + cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=women, REML = FALSE, 
                  control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelw2h)

modelw3h <- lmer(zflu ~ cage + cagesquared  + practice +
                   factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                   chdi + cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=women, REML = FALSE, 
                 control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelw3h)

```
### Men, controlling for hdi
```
modelm1h <- lmer(zimm ~ cage + cagesquared  + practice +
                   factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                   chdi + cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=men, REML = FALSE, 
                 control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelm1h)

modelm2h <- lmer(zdel ~ cage + cagesquared  + practice +
                   factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                   chdi + cieo + cieo*cage + (1+cage | mergeid) + (1 | cohcoun), data=men, REML = FALSE, 
                 control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelm2h)

modelm3h <- lmer(zflu ~ cage + cagesquared  + practice +
                   factor(edu) + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                   chdi + cieo + cieo*cage + (1 | mergeid) + (1 | cohcoun), data=men, REML = FALSE, 
                 control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelm3h)
tab_model(modelw1h,modelw2h,modelw3h,modelm1h,modelm2h,modelm3h, show.est=FALSE, show.std="std", collapse.ci=TRUE, p.style =c("asterisk"), file = "your global path/TableHDI.doc")
```
Further sets of analyses plus additional individual-level covariates (childhood variables, ISCO information), and plus GDP and HALE as additional contextual determinants.

### Interaction
```
modelboth1a <- lmer(zimm ~ cage + cagesquared  + practice + male +
                     factor(edu) + + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                     cieo + cieo*male + cieo*factor(edu) + male*factor(edu) + cieo*male*factor(edu) + (1 | mergeid) + (1 | cohcoun), data=both, REML = FALSE, 
                   control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelboth1a)
modelboth1b <- lmer(zdel ~ cage + cagesquared  + practice + male +
                      factor(edu) + + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                      cieo + cieo*male + cieo*factor(edu) + male*factor(edu) + cieo*male*factor(edu) + (1 | mergeid) + (1 | cohcoun), data=both, REML = FALSE, 
                    control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelboth1b)
modelboth1c <- lmer(zflu ~ cage + cagesquared  + practice + male +
                      factor(edu) + + factor(currentsit) + living + factor(srh) + chronic + depressed + 
                      cieo + cieo*male + cieo*factor(edu) + male*factor(edu) + cieo*male*factor(edu) + (1 | mergeid) + (1 | cohcoun), data=both, REML = FALSE, 
                    control = lmerControl(optimizer ='optimx', optCtrl=list(method='nlminb')), na.action = na.exclude)
summary(modelboth1c)
tab_model(modelboth1a, modelboth1b,modelboth1c, show.est=FALSE, show.std="std", collapse.ci=TRUE, p.style =c("asterisk"), file = "your global path/TableINTERACTION.doc")
  ```
## Figure 1
`tiff` command to have 300dpi.
```
tiff("test2.tiff", units="in", width=3.2, height=3.2, res=300)
ggplot(data=diff2, aes(x=ieo, y=diff_ctot, size = sample)) +
  viridis::scale_color_viridis(discrete = TRUE)+
  geom_point(color = "darkblue") +
  coord_fixed(ratio=0.6, expand=TRUE)+
  geom_smooth(method=lm,se=FALSE, size=2, alpha=.8, color= "blue")+theme_minimal()+
  labs(y = "Female advantage in cognition", x = "IEO", size = "Sample size")+
  theme(legend.position = "none")+
  ggsave(filename = "your global path/figure1.png")
dev.off()
```






