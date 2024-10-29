Stata Do-file:
//We first start by calculating a unique observation id. This is done so that Stata recognizesthe dates in the dataset. We create a unique id because the command tsset, which is used toset the time variable, does not work for repeated values.

egen obs_id = seq(), from(1)
tsset obs_id

//here we start the process of calculating the CSAD values, which begins with calculating thehourly returns of each cryptocurrency
gen btc_return = (btcusdclose / l1.btcusdclose) - 1
gen eth_return = (ethusdclose / l1.ethusdclose) - 1
gen ltc_return = (ltcusdclose / l1.ltcusdclose) - 1
gen xrp_return = (xrpusdclose / l1.xrpusdclose) - 1
gen btc_return = (btceurclose / l1.btceurclose) - 1
gen eth_return = (etheurclose / l1.etheurclose) - 1
gen ltc_return = (ltceurclose / l1.ltceurclose) - 1
gen xrp_return = (xrpeurclose / l1.xrpeurclose) - 1

// calculate the average return of the equally weighted portfolio at each time point (repeat the process for the European values)

egen avg_return = rowmean(btc_return eth_return ltc_return xrp_return)

//calculate the CSAD for each cryptocurrency (repeat for european values)

gen abs_dev_btc = abs(btc_return - avg_return)
gen abs_dev_eth = abs(eth_return - avg_return)
gen abs_dev_ltc = abs(ltc_return - avg_return)
gen abs_dev_xrp = abs(xrp_return - avg_return)

//calculate the mean CSAD (repeat the process for the european CSAD value)

egen CSAD_usa = rowmean(abs_dev_btc abs_dev_eth abs_dev_ltc abs_dev_xrp)

// after loading the case data, we manually remove all the data that is not useful to us, andthen we aggregate the case data into a single variable for daily cases we do this using a loop.We only need to do this for the Europe data because the USA data is already aggregated for us.

egen europe_cases = rowtotal(germany_cases france_cases spain_cases netherlands_cases italy_cases)

//now we expand it to fit the hourly price data
expand 24
// the event variable is created manually without code in an excel file
egen event_variable = .
// normalizing the variables using z-scores so that they are on the same scale 

// normalizing the variables using z-scores so that they are on the same scale
foreach var of varlist csad_usa cases_usa {
 summarize `var'

 scalar `var'_mean = r(mean)
 scalar `var'_sd = r(sd)
}
// Calculate z-scores for the variables

foreach var of varlist csad_usa cases_usa {
 gen `var'_z = (`var' - `var'_mean) / `var'_sd
}
//plotting the variables

tsline norm_csad_usa norm_cases_usa

//we repeat the same for the european csad and cases variables
// We start with the Markov-Switching Model

mswitch dr norm_csad norm_cases_usa event_dummy, vce(robust)

// calculate the residuals

predict residuals, residuals

// We calculate the AIC

estat ic

// ACF and PACF plots

ac residuals
pac residuals

// Calculate the Ljung-Box statistic for heteroskedasticity

wntestq residuals

// VAR model where we repeat the same process with the M-switch model, we also do the same with subsequent var models and the ARIMA models

var norm_csad, lags(1/80) exog(norm_cases_usa event_dummy)
predict residuals, residuals
ac residuals
pac residuals
estat ic
wntestq residuals

// ARIMA model

arima norm_csad norm_cases_usa event_dummy, arima(1,0,1) vce(robust)
predict residuals, residuals
ac residuals
pac residuals
estat ic
wntestq residuals

// we now move to forecasting, we restimate the model and then predic the next 29500 values

var norm_csad, lags(1/80) exog(norm_cases_usa event_dummy)

//forecast the values and place them into a new variable

tempfile forecasts
forvalues i = 1(500)30000 {
 predict forecasted_norm_csad`i' if _n > 22 & _n <= (`i' + 22)
 save `forecasts'_`i', replace
}

//we calculate the MAE, MSE, and RMSE measures. this process, including the forecasting isrepeated and each model's measures are compared with one another

summarize actual_variable forecasted_variable, meanonly
di "MAE: " abs(r(mean))

// Compute absolute differences

gen abs_diff = abs(actual_variable - forecasted_variable)

// Compute squared differences

gen squared_diff = abs_diff^2

// Calculate MSE

sum squared_diff, meanonly

di "MSE: " r(mean)

// Calculate RMSE

di "RMSE: " sqrt(`mse')

//plot the forecasted variable as well as the original variable

tsline forecasted_norm_csad_usa norm_csad_usa

// we then summarize each mean for the forecasted period, the real period, and the pre-period to compare mean csad values

sum forecasted_norm_csad_usa
sum norm_csad_usa
sum pre_norm_csad_usa

// end do-file
Supplementary files available upon request:
USA Dataset (PP)
USA Dataset (PPP)
Europe Dataset (PP)
Europe Dataset (PPP)
Forecast Data (USA)
Forecast Data (Europe)