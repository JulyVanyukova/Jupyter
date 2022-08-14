Найдем самые лучшие из степенных факторов используя модель линейной регрессии
import optuna
import numpy as np
import pandas as pd
from sklearn.preprocessing import MinMaxScaler
from sklearn.linear_model import LinearRegression, ElasticNet
from sklearn.isotonic import IsotonicRegression
from sklearn.metrics import mean_squared_log_error, mean_squared_error
from sklearn.model_selection import train_test_split
data = pd.read_csv('energy_2.csv')
print(data.head())
             timestamp  meter_reading  air_temperature  cloud_coverage  \
0  2016-01-30 08:00:00        43.6839              8.3             0.0   
1  2016-01-31 05:00:00        37.5408             12.8             0.0   
2  2016-01-31 17:00:00        52.5571             20.6             0.0   
3  2016-04-08 14:00:00        59.3827             21.7             2.0   
4  2016-05-01 19:00:00       448.0000             31.1             0.0   

   dew_temperature  precip_depth_1_hr  sea_level_pressure  wind_speed  \
0              6.1                0.0              1019.0         2.1   
1             10.0                0.0              1021.9         0.0   
2             11.7                0.0              1020.9         1.5   
3             14.4                0.0              1015.1         3.1   
4             17.2                0.0              1016.1         4.1   

   wind_direction_sin  wind_direction_cos  air_temperature1  hour  
0           -0.642788           -0.766044              -2.3     8  
1            0.000000            1.000000              -1.1     5  
2            0.939693           -0.342020               1.7    17  
3           -0.939693           -0.342020               2.8    14  
4            0.984808           -0.173648               1.1    19  
Подбор параметров
def calculate_bic(y, y_pred, power):
    return len(y)*np.log(len(y)*mean_squared_error(y, y_pred)**2) + power*np.log(len(y))
columns=list(data.columns)
columns.remove('timestamp')
columns.remove('meter_reading')
columns.remove('hour')
Линейные значения
best_columns = []
current_columns = []
power = 0
best_power = 0
bic_best = 10000000
for column in columns:
    current_columns.append(column)
    power += 1
    y = data['meter_reading']
    x = MinMaxScaler().fit_transform(data[current_columns])
    bic= calculate_bic(y, LinearRegression().fit(x, y).predict(x), power)
    if bic < bic_best:
        best_columns.append(column)
        best_power += 1
        bic_best = bic
    else:
        power -= 1
        current_columns.remove(column)
print(best_columns)        
    
['air_temperature', 'dew_temperature', 'precip_depth_1_hr', 'sea_level_pressure', 'wind_speed', 'air_temperature1']
Квадратичное значение
for column in columns:
    data[column +'_2'] = np.multiply(data[column], data[column])
for column in columns:
    c = column + '_2'
    current_columns.append(c)
    power += 1
    y = data['meter_reading']
    x = MinMaxScaler().fit_transform(data[current_columns])
    bic= calculate_bic(y, LinearRegression().fit(x, y).predict(x), power)
    if bic < bic_best:
        best_columns.append(c)
        best_power += 1
        bic_best = bic
    else:
        power -= 1
        current_columns.remove(c)
print(best_columns)        
    
['air_temperature', 'dew_temperature', 'precip_depth_1_hr', 'sea_level_pressure', 'wind_speed', 'air_temperature1', 'air_temperature_2', 'cloud_coverage_2', 'dew_temperature_2', 'sea_level_pressure_2', 'wind_speed_2']
Кубические значения
for column in columns:
    data [column + '_3']=np.multiply(data[column + '_2'], data[column])
for column in columns:
    c = column + '_3'
    current_columns.append(c)
    power += 1
    y = data['meter_reading']
    x = MinMaxScaler().fit_transform(data[current_columns])
    bic= calculate_bic(y, LinearRegression().fit(x, y).predict(x), power)
    if bic < bic_best:
        best_columns.append(c)
        best_power += 1
        bic_best = bic
    else:
        power -= 1
        current_columns.remove(c)
print(best_columns)   
['air_temperature', 'dew_temperature', 'precip_depth_1_hr', 'sea_level_pressure', 'wind_speed', 'air_temperature1', 'air_temperature_2', 'cloud_coverage_2', 'dew_temperature_2', 'sea_level_pressure_2', 'wind_speed_2', 'air_temperature_3', 'cloud_coverage_3', 'sea_level_pressure_3', 'wind_direction_sin_3', 'wind_direction_cos_3']
data_norm = pd.DataFrame(MinMaxScaler().fit_transform(data[best_columns]))
Разделение данных на обучающие и проверочные модели
train, test, y_train, y_test = train_test_split(data_norm, data['meter_reading'], test_size=0.2)
Модели регрессии
Линейной и изитерической регрессии

def rmsle_err(y, y_pred):
    return((np.log(1+y)-np.log(1+y_pred))**2).mean()**0.5
y = data['meter_reading']
model1 = LinearRegression().fit(train, y_train)
print('RMSLE:{0:.5}'.format(rmsle_err(y_train, model1.predict(train))))
RMSLE:0.19566
model2 = IsotonicRegression(out_of_bounds='clip').fit(train[0], y_train)
print('RMSLE:{0:.5}'.format(rmsle_err(y_train, model2.predict(train[0]))))
RMSLE:0.21745
ElasticNet
Выберем наилучшие гиперпараметры модели

def objective_elastic(trial):
    alpha = trial.suggest_float('alpha', 1e-8, 1, log=True)
    l1_ratio = trial.suggest_float('l1_ratio', 1e-3, 1, log=True)
    regressor_obj = ElasticNet(alpha=alpha, l1_ratio=l1_ratio, max_iter = 10000, tol = 1)
    regressor_obj.fit(train, y_train)
    y_pred= regressor_obj.predict(train)
    return mean_squared_log_error(y_train, y_pred)
study_elastic = optuna.create_study()
study_elastic.optimize(objective_elastic, n_trials=100)
[I 2022-06-19 09:42:49,641] A new study created in memory with name: no-name-af542709-6269-4c0b-9def-0d958034d58c
[I 2022-06-19 09:42:49,855] Trial 0 finished with value: 0.04266637762192197 and parameters: {'alpha': 0.0009732111743282171, 'l1_ratio': 0.40602640279871466}. Best is trial 0 with value: 0.04266637762192197.
[I 2022-06-19 09:42:49,870] Trial 1 finished with value: 0.042704963998789036 and parameters: {'alpha': 0.0002699826999467476, 'l1_ratio': 0.02403653333965759}. Best is trial 0 with value: 0.04266637762192197.
[I 2022-06-19 09:42:49,882] Trial 2 finished with value: 0.04273784639424791 and parameters: {'alpha': 1.4810342036706705e-07, 'l1_ratio': 0.001916048913192047}. Best is trial 0 with value: 0.04266637762192197.
[I 2022-06-19 09:42:49,892] Trial 3 finished with value: 0.04273785814178568 and parameters: {'alpha': 5.5972652470358775e-08, 'l1_ratio': 0.016221553352145042}. Best is trial 0 with value: 0.04266637762192197.
[I 2022-06-19 09:42:49,916] Trial 4 finished with value: 0.042737862991840556 and parameters: {'alpha': 1.7663021510190557e-08, 'l1_ratio': 0.05042857417523947}. Best is trial 0 with value: 0.04266637762192197.
[I 2022-06-19 09:42:49,941] Trial 5 finished with value: 0.041211897783741104 and parameters: {'alpha': 0.023184751400879455, 'l1_ratio': 0.002128742235059358}. Best is trial 5 with value: 0.041211897783741104.
[I 2022-06-19 09:42:49,957] Trial 6 finished with value: 0.041208578832663566 and parameters: {'alpha': 0.02269852675074695, 'l1_ratio': 0.010537407707877236}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:49,981] Trial 7 finished with value: 0.042588301860327665 and parameters: {'alpha': 0.001778769391771177, 'l1_ratio': 0.31016084300009544}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,005] Trial 8 finished with value: 0.04269455043399228 and parameters: {'alpha': 0.00035220318221200323, 'l1_ratio': 0.011348120066067606}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,028] Trial 9 finished with value: 0.04250799426237754 and parameters: {'alpha': 0.001905401801244716, 'l1_ratio': 0.005647034402560137}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,118] Trial 10 finished with value: 0.048720216133044725 and parameters: {'alpha': 0.1866871043028705, 'l1_ratio': 0.095177192305019}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,148] Trial 11 finished with value: 0.05484875608232762 and parameters: {'alpha': 0.49316163229651666, 'l1_ratio': 0.0014155172852360768}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,182] Trial 12 finished with value: 0.042207014828495905 and parameters: {'alpha': 0.046448950806659846, 'l1_ratio': 0.003938270050177469}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,215] Trial 13 finished with value: 0.04273761630212969 and parameters: {'alpha': 1.9752347831565476e-06, 'l1_ratio': 0.005370664697727884}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,247] Trial 14 finished with value: 0.04131551564462644 and parameters: {'alpha': 0.01561992979296114, 'l1_ratio': 0.0010314979511808146}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,285] Trial 15 finished with value: 0.042736429614598274 and parameters: {'alpha': 1.1374161198500144e-05, 'l1_ratio': 0.002867162272743582}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,316] Trial 16 finished with value: 0.04124529880740739 and parameters: {'alpha': 0.02588399584467028, 'l1_ratio': 0.00830191499101547}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,352] Trial 17 finished with value: 0.042735135168864805 and parameters: {'alpha': 2.2614510921927735e-05, 'l1_ratio': 0.045920143642160814}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,388] Trial 18 finished with value: 0.04887527539256148 and parameters: {'alpha': 0.802129056719149, 'l1_ratio': 0.8415497006357281}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,421] Trial 19 finished with value: 0.041986450578376605 and parameters: {'alpha': 0.006367089743518911, 'l1_ratio': 0.0026136518537706144}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,457] Trial 20 finished with value: 0.0455322068849721 and parameters: {'alpha': 0.11218791943361654, 'l1_ratio': 0.12664565456215932}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,489] Trial 21 finished with value: 0.04124198835249858 and parameters: {'alpha': 0.018328681014510653, 'l1_ratio': 0.00886781736428734}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,528] Trial 22 finished with value: 0.042103200684533756 and parameters: {'alpha': 0.005363954517249301, 'l1_ratio': 0.012814887422413577}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,586] Trial 23 finished with value: 0.044828394760148216 and parameters: {'alpha': 0.08837978506116836, 'l1_ratio': 0.021599299818498677}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,628] Trial 24 finished with value: 0.0427307006097563 and parameters: {'alpha': 5.7145523879477474e-05, 'l1_ratio': 0.006725177181389142}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,663] Trial 25 finished with value: 0.04187119146672535 and parameters: {'alpha': 0.007487430296713553, 'l1_ratio': 0.003384628976890441}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,704] Trial 26 finished with value: 0.051764919634334355 and parameters: {'alpha': 0.29248167761932803, 'l1_ratio': 0.039112654409829674}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,745] Trial 27 finished with value: 0.04155371474690638 and parameters: {'alpha': 0.034995521522667605, 'l1_ratio': 0.008946129598641014}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,782] Trial 28 finished with value: 0.0426940349441444 and parameters: {'alpha': 0.00035305803438843064, 'l1_ratio': 0.00178507381302809}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,815] Trial 29 finished with value: 0.042508463419751875 and parameters: {'alpha': 0.0020446042792997684, 'l1_ratio': 0.07522416499787397}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,852] Trial 30 finished with value: 0.04265286029341092 and parameters: {'alpha': 0.0007034579864498029, 'l1_ratio': 0.017457712462099952}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,887] Trial 31 finished with value: 0.04133022530905069 and parameters: {'alpha': 0.015351776364183995, 'l1_ratio': 0.008347037668905249}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,930] Trial 32 finished with value: 0.04123337187436684 and parameters: {'alpha': 0.01869313218412245, 'l1_ratio': 0.0043001094873651256}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:50,973] Trial 33 finished with value: 0.04428833043743697 and parameters: {'alpha': 0.07811333212315458, 'l1_ratio': 0.0038134795755370232}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,017] Trial 34 finished with value: 0.04209347103925525 and parameters: {'alpha': 0.005392337619502321, 'l1_ratio': 0.002143482454487647}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,064] Trial 35 finished with value: 0.04273774256845641 and parameters: {'alpha': 9.93764247443781e-07, 'l1_ratio': 0.026496563626220285}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,113] Trial 36 finished with value: 0.050681423744130684 and parameters: {'alpha': 0.23635196587924234, 'l1_ratio': 0.004778027197294876}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,154] Trial 37 finished with value: 0.04272862424654637 and parameters: {'alpha': 7.428376128752824e-05, 'l1_ratio': 0.013518276018642875}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,193] Trial 38 finished with value: 0.04248063584309082 and parameters: {'alpha': 0.002123400536601284, 'l1_ratio': 0.0011044039584434363}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,268] Trial 39 finished with value: 0.04131130328546205 and parameters: {'alpha': 0.015741523894842912, 'l1_ratio': 0.0015334024320678916}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,306] Trial 40 finished with value: 0.05782024316640771 and parameters: {'alpha': 0.9763174628554662, 'l1_ratio': 0.0024455553076538775}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,347] Trial 41 finished with value: 0.0413647730015021 and parameters: {'alpha': 0.030347324338235504, 'l1_ratio': 0.008518836732746372}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,381] Trial 42 finished with value: 0.042610269428146524 and parameters: {'alpha': 0.0010523560687009095, 'l1_ratio': 0.007064797153962375}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,419] Trial 43 finished with value: 0.04156862905848737 and parameters: {'alpha': 0.03537447027310644, 'l1_ratio': 0.010751866138839168}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,455] Trial 44 finished with value: 0.04149436258553388 and parameters: {'alpha': 0.012278225705994118, 'l1_ratio': 0.018144897423340356}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,499] Trial 45 finished with value: 0.04235569418560861 and parameters: {'alpha': 0.003167873480556174, 'l1_ratio': 0.004543514825881202}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,539] Trial 46 finished with value: 0.04614546510233723 and parameters: {'alpha': 0.11340616647849402, 'l1_ratio': 0.030582547252120156}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,585] Trial 47 finished with value: 0.04264591168355699 and parameters: {'alpha': 0.0007513333989702678, 'l1_ratio': 0.0033976168520775327}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,633] Trial 48 finished with value: 0.042709897720988696 and parameters: {'alpha': 0.00022508301751138712, 'l1_ratio': 0.006687971009129367}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,682] Trial 49 finished with value: 0.053461841244550154 and parameters: {'alpha': 0.38128515522768797, 'l1_ratio': 0.005320175044471432}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,717] Trial 50 finished with value: 0.041241249628744266 and parameters: {'alpha': 0.025723298482246477, 'l1_ratio': 0.01117178873422019}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,757] Trial 51 finished with value: 0.04121612431614297 and parameters: {'alpha': 0.023881068761609333, 'l1_ratio': 0.012658225522143667}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,796] Trial 52 finished with value: 0.04407153290700167 and parameters: {'alpha': 0.07529388577294888, 'l1_ratio': 0.011871383176782862}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,832] Trial 53 finished with value: 0.04159460035179631 and parameters: {'alpha': 0.010841857047676576, 'l1_ratio': 0.02099500390890327}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,873] Trial 54 finished with value: 0.04840040477791245 and parameters: {'alpha': 0.163638243016621, 'l1_ratio': 0.014448853991854508}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,910] Trial 55 finished with value: 0.04218729193033632 and parameters: {'alpha': 0.04643593440161262, 'l1_ratio': 0.01070346140149986}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,947] Trial 56 finished with value: 0.042328630422477885 and parameters: {'alpha': 0.0033970638673329655, 'l1_ratio': 0.005654879936095732}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:51,997] Trial 57 finished with value: 0.0412948835104175 and parameters: {'alpha': 0.02076391022226777, 'l1_ratio': 0.216187770906374}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:52,026] Trial 58 finished with value: 0.04273786315238208 and parameters: {'alpha': 1.55536315314e-08, 'l1_ratio': 0.0027760501409003296}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:52,060] Trial 59 finished with value: 0.04180892863577942 and parameters: {'alpha': 0.00841742959351134, 'l1_ratio': 0.037792544394906806}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:52,092] Trial 60 finished with value: 0.04273785529129642 and parameters: {'alpha': 8.202857631404019e-08, 'l1_ratio': 0.05464337473463807}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:52,119] Trial 61 finished with value: 0.04121047677696191 and parameters: {'alpha': 0.023074895348055017, 'l1_ratio': 0.009238532193526732}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:52,167] Trial 62 finished with value: 0.0425639308097879 and parameters: {'alpha': 0.05211043439650081, 'l1_ratio': 0.006703362598999638}. Best is trial 6 with value: 0.041208578832663566.
[I 2022-06-19 09:42:52,212] Trial 63 finished with value: 0.04120708901376989 and parameters: {'alpha': 0.021827574680803263, 'l1_ratio': 0.009712751066008166}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,259] Trial 64 finished with value: 0.04812466930944579 and parameters: {'alpha': 0.15658167506140183, 'l1_ratio': 0.01619928991250686}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,305] Trial 65 finished with value: 0.04230383013791121 and parameters: {'alpha': 0.0035991339029137217, 'l1_ratio': 0.004182281767215782}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,347] Trial 66 finished with value: 0.04298951370718482 and parameters: {'alpha': 0.05869963274804215, 'l1_ratio': 0.010042458786488183}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,376] Trial 67 finished with value: 0.041208603595563616 and parameters: {'alpha': 0.022613779349963777, 'l1_ratio': 0.0020572662227117434}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,414] Trial 68 finished with value: 0.04259130366321019 and parameters: {'alpha': 0.001204303271577667, 'l1_ratio': 0.0012897297468303552}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,448] Trial 69 finished with value: 0.04202273026032375 and parameters: {'alpha': 0.006035257050390529, 'l1_ratio': 0.0031914792059749312}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,482] Trial 70 finished with value: 0.041642847462154436 and parameters: {'alpha': 0.010017594606378945, 'l1_ratio': 0.0021724694986566886}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,514] Trial 71 finished with value: 0.04128094629190705 and parameters: {'alpha': 0.027368101213267493, 'l1_ratio': 0.0019590379557999547}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,549] Trial 72 finished with value: 0.05533017870914297 and parameters: {'alpha': 0.5431990419105206, 'l1_ratio': 0.0016006822937318379}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,578] Trial 73 finished with value: 0.041333740652067 and parameters: {'alpha': 0.01524785112733013, 'l1_ratio': 0.007276888349892989}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,614] Trial 74 finished with value: 0.04618299612711814 and parameters: {'alpha': 0.11141973794680923, 'l1_ratio': 0.00536877221270416}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,646] Trial 75 finished with value: 0.04122189633554429 and parameters: {'alpha': 0.02468185928016926, 'l1_ratio': 0.02514307602998829}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,677] Trial 76 finished with value: 0.042736719260170725 and parameters: {'alpha': 9.285050254581765e-06, 'l1_ratio': 0.025267218293112636}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,712] Trial 77 finished with value: 0.04239234424016435 and parameters: {'alpha': 0.05008724860782494, 'l1_ratio': 0.019337672347416457}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,746] Trial 78 finished with value: 0.04225003510676584 and parameters: {'alpha': 0.004089151757758091, 'l1_ratio': 0.013374946809376585}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,782] Trial 79 finished with value: 0.05160006615528664 and parameters: {'alpha': 0.274778736735875, 'l1_ratio': 0.0012790204647019606}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,817] Trial 80 finished with value: 0.04184357058275815 and parameters: {'alpha': 0.007860663960498754, 'l1_ratio': 0.015199260020998522}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,852] Trial 81 finished with value: 0.04121599100914617 and parameters: {'alpha': 0.023816391463486884, 'l1_ratio': 0.009900304477429383}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,892] Trial 82 finished with value: 0.04121636233085171 and parameters: {'alpha': 0.0238362780714029, 'l1_ratio': 0.008815973002679191}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,932] Trial 83 finished with value: 0.04152697847195461 and parameters: {'alpha': 0.03612895467754791, 'l1_ratio': 0.05931198341616585}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:52,985] Trial 84 finished with value: 0.04121053732253819 and parameters: {'alpha': 0.023294934559400824, 'l1_ratio': 0.024020685775896798}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,040] Trial 85 finished with value: 0.04369528618336167 and parameters: {'alpha': 0.06930745639894939, 'l1_ratio': 0.009458690920808975}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,090] Trial 86 finished with value: 0.04151977610400945 and parameters: {'alpha': 0.012047283060859098, 'l1_ratio': 0.03171934204466602}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,123] Trial 87 finished with value: 0.04242161141417335 and parameters: {'alpha': 0.002629258548627443, 'l1_ratio': 0.00784280130423969}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,160] Trial 88 finished with value: 0.04254788746674707 and parameters: {'alpha': 0.0015910126782436324, 'l1_ratio': 0.017141226006067503}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,191] Trial 89 finished with value: 0.04623287690903111 and parameters: {'alpha': 0.1131301625436459, 'l1_ratio': 0.012174539703886827}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,224] Trial 90 finished with value: 0.04122259164951242 and parameters: {'alpha': 0.019425086977104003, 'l1_ratio': 0.005893480037818458}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,258] Trial 91 finished with value: 0.04155856004507194 and parameters: {'alpha': 0.035524577347526114, 'l1_ratio': 0.02160679215551712}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,293] Trial 92 finished with value: 0.04213069782759495 and parameters: {'alpha': 0.005194150637452607, 'l1_ratio': 0.026833114007695964}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,326] Trial 93 finished with value: 0.04120815296585569 and parameters: {'alpha': 0.02123478408709621, 'l1_ratio': 0.008604803270169317}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,359] Trial 94 finished with value: 0.04144443140590075 and parameters: {'alpha': 0.012994030798579586, 'l1_ratio': 0.008822698368887204}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,389] Trial 95 finished with value: 0.044797235587484364 and parameters: {'alpha': 0.08721437370333603, 'l1_ratio': 0.014077966034541534}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,424] Trial 96 finished with value: 0.04954697629918267 and parameters: {'alpha': 0.19621192127267015, 'l1_ratio': 0.0062700984818628664}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,455] Trial 97 finished with value: 0.04189047543669353 and parameters: {'alpha': 0.00729771820505522, 'l1_ratio': 0.0037855635659617853}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,489] Trial 98 finished with value: 0.04166529459982707 and parameters: {'alpha': 0.03723493845992127, 'l1_ratio': 0.0076222852294489905}. Best is trial 63 with value: 0.04120708901376989.
[I 2022-06-19 09:42:53,554] Trial 99 finished with value: 0.04120742584365313 and parameters: {'alpha': 0.022292718577325697, 'l1_ratio': 0.009784783227184868}. Best is trial 63 with value: 0.04120708901376989.
model3 = ElasticNet(alpha=study_elastic.best_params['alpha'],
                   l1_ratio=study_elastic.best_params['l1_ratio'],
                   max_iter = 100000, tol=0.01).fit(train, y_train)
print('RMSLE:{0:.5}'.format(rmsle_err(y_train, model3.predict(train))))
RMSLE:0.20281
Объедиение моделей
Используем оптуна для поиска оптимального коэффециента
def objective(trial):
    alpha = trial.suggest_float('alpha', 1e-10, 1, log=True)
    beta = trial.suggest_float('beta', 1e-10, 1-alpha, log=True)
    y_pred= (alpha*model1.predict(test)+
            beta*model2.predict(test[0])+
            (1-alpha-beta)*model3.predict(test))
    return mean_squared_log_error(y_test, y_pred)
study = optuna.create_study()
study.optimize(objective,n_trials=100)
[I 2022-06-19 10:28:15,157] A new study created in memory with name: no-name-1f082ae1-ac27-4856-b926-4d7f5b44468e
[I 2022-06-19 10:28:15,174] Trial 0 finished with value: 0.04952701227962745 and parameters: {'alpha': 7.896163493507556e-05, 'beta': 1.0797980427792678e-10}. Best is trial 0 with value: 0.04952701227962745.
[I 2022-06-19 10:28:15,188] Trial 1 finished with value: 0.04758981945989846 and parameters: {'alpha': 0.2345985927840193, 'beta': 0.0027515402106223033}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,205] Trial 2 finished with value: 0.04948828410327212 and parameters: {'alpha': 0.004343120313333712, 'beta': 5.328927776028681e-08}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,223] Trial 3 finished with value: 0.049527443211617894 and parameters: {'alpha': 5.449608524805599e-10, 'beta': 0.00011351683064474555}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,249] Trial 4 finished with value: 0.049526081714442644 and parameters: {'alpha': 0.00016831209981599, 'beta': 4.645056183143507e-05}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,270] Trial 5 finished with value: 0.04952534345257946 and parameters: {'alpha': 0.0002623959040470997, 'beta': 4.193442151977898e-09}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,288] Trial 6 finished with value: 0.04952773021872685 and parameters: {'alpha': 5.0447813046376397e-08, 'beta': 1.735491536661391e-08}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,304] Trial 7 finished with value: 0.04952773039253566 and parameters: {'alpha': 2.2030767884138664e-08, 'beta': 5.0813126434451563e-08}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,323] Trial 8 finished with value: 0.04952726846743823 and parameters: {'alpha': 5.0795333462658044e-05, 'beta': 3.2172106243653315e-08}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,341] Trial 9 finished with value: 0.049527725528321166 and parameters: {'alpha': 1.0842975904381442e-08, 'beta': 2.0112090919846635e-06}. Best is trial 1 with value: 0.04758981945989846.
[I 2022-06-19 10:28:15,376] Trial 10 finished with value: 0.04646995612487122 and parameters: {'alpha': 0.3543514199878227, 'beta': 0.4422405756456081}. Best is trial 10 with value: 0.04646995612487122.
[I 2022-06-19 10:28:15,399] Trial 11 finished with value: 0.04412748403709483 and parameters: {'alpha': 0.9122822534168237, 'beta': 0.027217352296792355}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,437] Trial 12 finished with value: 0.04760262217976061 and parameters: {'alpha': 0.25079969054547036, 'beta': 0.6639907056864445}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,469] Trial 13 finished with value: 0.04912452636240217 and parameters: {'alpha': 0.016481176890354332, 'beta': 0.5077153879882037}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,508] Trial 14 finished with value: 0.04418765238864441 and parameters: {'alpha': 0.9074633985980305, 'beta': 0.007320968035148648}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,541] Trial 15 finished with value: 0.04944661539756012 and parameters: {'alpha': 0.007072207815971378, 'beta': 0.006772584436041567}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,570] Trial 16 finished with value: 0.04950906677656285 and parameters: {'alpha': 1.2279544329884335e-06, 'beta': 0.0074535022134772436}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,605] Trial 17 finished with value: 0.0444459855433542 and parameters: {'alpha': 0.8268725139254126, 'beta': 0.0005903792198962034}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,637] Trial 18 finished with value: 0.04911864528481631 and parameters: {'alpha': 0.03424697350422986, 'beta': 0.04343346791961927}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,669] Trial 19 finished with value: 0.049527683770298744 and parameters: {'alpha': 3.2283293921560963e-06, 'beta': 6.938860321698831e-06}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,713] Trial 20 finished with value: 0.049443095425077806 and parameters: {'alpha': 0.0016166091981410678, 'beta': 0.028972661075749466}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,747] Trial 21 finished with value: 0.04646850857458047 and parameters: {'alpha': 0.40051532716953175, 'beta': 0.00035899286930254963}. Best is trial 11 with value: 0.04412748403709483.
[I 2022-06-19 10:28:15,780] Trial 22 finished with value: 0.04411237409540404 and parameters: {'alpha': 0.94272505661626, 'beta': 0.0007819425373725758}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:15,812] Trial 23 finished with value: 0.04905062230427919 and parameters: {'alpha': 0.03909591382202015, 'beta': 0.05537555823179122}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:15,848] Trial 24 finished with value: 0.04899886105687599 and parameters: {'alpha': 0.0592009017283222, 'beta': 0.0013392702891535369}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:15,885] Trial 25 finished with value: 0.04872986040636757 and parameters: {'alpha': 0.091092160500724, 'beta': 3.7416823956055e-05}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:15,921] Trial 26 finished with value: 0.04935121806397212 and parameters: {'alpha': 0.0014917424698909498, 'beta': 0.07292715881338034}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:15,957] Trial 27 finished with value: 0.04501788629769449 and parameters: {'alpha': 0.6765160177632477, 'beta': 3.404880017961718e-06}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:15,988] Trial 28 finished with value: 0.04949903315296981 and parameters: {'alpha': 0.0008959994566253615, 'beta': 0.00822198647123309}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,021] Trial 29 finished with value: 0.049527580080037215 and parameters: {'alpha': 1.6413531039895034e-05, 'beta': 5.121391449109245e-07}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,053] Trial 30 finished with value: 0.04942557164724751 and parameters: {'alpha': 0.011242055997198597, 'beta': 0.00013651041773303464}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,088] Trial 31 finished with value: 0.044216747081007826 and parameters: {'alpha': 0.9027833599215258, 'beta': 0.0007651126317257236}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,133] Trial 32 finished with value: 0.04880970057793587 and parameters: {'alpha': 0.0811843739538576, 'beta': 0.001610304965400966}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,175] Trial 33 finished with value: 0.04429938510314749 and parameters: {'alpha': 0.8715116758142223, 'beta': 0.003482857149537932}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,203] Trial 34 finished with value: 0.047955828983333525 and parameters: {'alpha': 0.15175510226991068, 'beta': 0.14558814469841833}. Best is trial 22 with value: 0.04411237409540404.
[I 2022-06-19 10:28:16,232] Trial 35 finished with value: 0.044105300485098385 and parameters: {'alpha': 0.9459122701442039, 'beta': 0.00047573738498510895}. Best is trial 35 with value: 0.044105300485098385.
[I 2022-06-19 10:28:16,274] Trial 36 finished with value: 0.04947871434586484 and parameters: {'alpha': 0.005387198244727736, 'beta': 4.276208369971764e-05}. Best is trial 35 with value: 0.044105300485098385.
[I 2022-06-19 10:28:16,314] Trial 37 finished with value: 0.04952773071839178 and parameters: {'alpha': 2.7157386557227454e-10, 'beta': 3.3457934409187343e-10}. Best is trial 35 with value: 0.044105300485098385.
[I 2022-06-19 10:28:16,348] Trial 38 finished with value: 0.048360840826181105 and parameters: {'alpha': 0.13574832229434053, 'beta': 0.0001736564713853737}. Best is trial 35 with value: 0.044105300485098385.
[I 2022-06-19 10:28:16,386] Trial 39 finished with value: 0.04947635249904269 and parameters: {'alpha': 4.258760478972732e-07, 'beta': 0.020994901525970982}. Best is trial 35 with value: 0.044105300485098385.
[I 2022-06-19 10:28:16,427] Trial 40 finished with value: 0.0493167404761194 and parameters: {'alpha': 0.023411508596038962, 'beta': 1.4265833194756116e-05}. Best is trial 35 with value: 0.044105300485098385.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:16,499] Trial 41 finished with value: 0.04399886071342916 and parameters: {'alpha': 0.9927873629065544, 'beta': 0.00018135989674655038}. Best is trial 41 with value: 0.04399886071342916.
[I 2022-06-19 10:28:16,536] Trial 42 finished with value: 0.04747860116001181 and parameters: {'alpha': 0.25061751961183676, 'beta': 0.00029070495100380796}. Best is trial 41 with value: 0.04399886071342916.
[I 2022-06-19 10:28:16,574] Trial 43 finished with value: 0.047896456835903926 and parameters: {'alpha': 0.1941878733988539, 'beta': 0.001715736654454108}. Best is trial 41 with value: 0.04399886071342916.
[I 2022-06-19 10:28:16,606] Trial 44 finished with value: 0.049500009864517844 and parameters: {'alpha': 1.540977451768629e-09, 'beta': 0.011144516000961472}. Best is trial 41 with value: 0.04399886071342916.
[I 2022-06-19 10:28:16,630] Trial 45 finished with value: 0.04656973626691967 and parameters: {'alpha': 0.38309805029363586, 'beta': 0.0036223898291748717}. Best is trial 41 with value: 0.04399886071342916.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:16,671] Trial 46 finished with value: 0.0439914548506695 and parameters: {'alpha': 0.996424180299072, 'beta': 7.658871790599735e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:16,720] Trial 47 finished with value: 0.049214541186097416 and parameters: {'alpha': 0.034924849713852915, 'beta': 7.561375764996926e-07}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:16,755] Trial 48 finished with value: 0.04873110325618418 and parameters: {'alpha': 0.09094037572961089, 'beta': 5.1445006141494336e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:16,792] Trial 49 finished with value: 0.04949425579471713 and parameters: {'alpha': 0.003677011055976688, 'beta': 2.7310659540999697e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:16,859] Trial 50 finished with value: 0.04921406266137514 and parameters: {'alpha': 7.979546633166745e-05, 'beta': 0.16984453203430316}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:16,914] Trial 51 finished with value: 0.046537518767920553 and parameters: {'alpha': 0.38958241690815654, 'beta': 8.655373425709158e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:16,959] Trial 52 finished with value: 0.04747286746430836 and parameters: {'alpha': 0.25127366106765187, 'beta': 0.0006804066993891113}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,002] Trial 53 finished with value: 0.045622863776504756 and parameters: {'alpha': 0.5489533897742868, 'beta': 0.00028140008911078967}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,045] Trial 54 finished with value: 0.04416925619460558 and parameters: {'alpha': 0.9068381320740181, 'beta': 0.01562675326287252}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,080] Trial 55 finished with value: 0.04935299170079648 and parameters: {'alpha': 0.019355686507792475, 'beta': 1.400901171943142e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,114] Trial 56 finished with value: 0.04812380357545016 and parameters: {'alpha': 0.1227073405214798, 'beta': 0.20728248016231887}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,145] Trial 57 finished with value: 0.048974925368713035 and parameters: {'alpha': 0.05732444905296523, 'beta': 0.017811490470337848}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,178] Trial 58 finished with value: 0.04726158120696093 and parameters: {'alpha': 0.27984922097563536, 'beta': 0.0033890770687260863}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,212] Trial 59 finished with value: 0.04410923483471271 and parameters: {'alpha': 0.9437265283388343, 'beta': 0.0010462784443974226}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,240] Trial 60 finished with value: 0.046323794892635994 and parameters: {'alpha': 0.4242307450655321, 'beta': 9.225213026888082e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,303] Trial 61 finished with value: 0.04412224880090427 and parameters: {'alpha': 0.9385687685924077, 'beta': 0.0009659777298468786}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,335] Trial 62 finished with value: 0.04825770006572812 and parameters: {'alpha': 0.14836891157118026, 'beta': 0.0008087612808756824}. Best is trial 46 with value: 0.0439914548506695.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:17,360] Trial 63 finished with value: 0.044042029240136754 and parameters: {'alpha': 0.9715419699132354, 'beta': 0.0015683030602507717}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,390] Trial 64 finished with value: 0.0441408255352843 and parameters: {'alpha': 0.9317582869190524, 'beta': 0.0004201248297612665}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,419] Trial 65 finished with value: 0.04894933904393513 and parameters: {'alpha': 0.06495449418140763, 'beta': 0.0012777708304294796}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,452] Trial 66 finished with value: 0.04952459002360047 and parameters: {'alpha': 0.000343429014125862, 'beta': 6.439440129379675e-06}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,480] Trial 67 finished with value: 0.04616231259289 and parameters: {'alpha': 0.45131404224923316, 'beta': 0.0001887341878150478}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,516] Trial 68 finished with value: 0.04938444435180777 and parameters: {'alpha': 0.014574001474446032, 'beta': 0.004572748086456471}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,543] Trial 69 finished with value: 0.047815451436723604 and parameters: {'alpha': 0.20475587191277864, 'beta': 0.0018587625050157804}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,574] Trial 70 finished with value: 0.04918142222096932 and parameters: {'alpha': 0.03865381917312269, 'beta': 8.795229596284593e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,606] Trial 71 finished with value: 0.04579536831155248 and parameters: {'alpha': 0.5163015184264389, 'beta': 0.0004837254705168952}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,639] Trial 72 finished with value: 0.044434953251748226 and parameters: {'alpha': 0.829886228158808, 'beta': 0.0010658311898733104}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,670] Trial 73 finished with value: 0.04756587246335923 and parameters: {'alpha': 0.23871433189416655, 'beta': 0.00017993157433172688}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,700] Trial 74 finished with value: 0.048633492557110046 and parameters: {'alpha': 0.10191682743599652, 'beta': 0.0023009613263368076}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,726] Trial 75 finished with value: 0.044126667966519095 and parameters: {'alpha': 0.9374846079960151, 'beta': 0.0002908007587010857}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,757] Trial 76 finished with value: 0.04608941729123707 and parameters: {'alpha': 0.4639188502642415, 'beta': 2.0749427206590938e-05}. Best is trial 46 with value: 0.0439914548506695.
[I 2022-06-19 10:28:17,786] Trial 77 finished with value: 0.04952702638234682 and parameters: {'alpha': 3.0590110974458305e-07, 'beta': 0.0002770717883804988}. Best is trial 46 with value: 0.0439914548506695.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:17,819] Trial 78 finished with value: 0.04398502464349977 and parameters: {'alpha': 0.9980386356688133, 'beta': 0.0012821821253618591}. Best is trial 78 with value: 0.04398502464349977.
[I 2022-06-19 10:28:18,003] Trial 79 finished with value: 0.0481217588388527 and parameters: {'alpha': 0.16396665912028674, 'beta': 0.0056763745107243635}. Best is trial 78 with value: 0.04398502464349977.
[I 2022-06-19 10:28:18,030] Trial 80 finished with value: 0.048957281629010425 and parameters: {'alpha': 0.06437684395627277, 'beta': 9.66911917599001e-05}. Best is trial 78 with value: 0.04398502464349977.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:18,060] Trial 81 finished with value: 0.043982397960851925 and parameters: {'alpha': 0.9921226833180047, 'beta': 0.007317691582675192}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,090] Trial 82 finished with value: 0.04584719565038929 and parameters: {'alpha': 0.5067304533914815, 'beta': 0.0005955495820533286}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,121] Trial 83 finished with value: 0.04723005281187489 and parameters: {'alpha': 0.2851671699251137, 'beta': 0.0009922850906513149}. Best is trial 81 with value: 0.043982397960851925.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:18,151] Trial 84 finished with value: 0.0439985398825913 and parameters: {'alpha': 0.9931365759386818, 'beta': 1.5908757682444025e-05}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,184] Trial 85 finished with value: 0.04568190216864872 and parameters: {'alpha': 0.5377614668069015, 'beta': 5.337473433030349e-05}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,205] Trial 86 finished with value: 0.048286175047284885 and parameters: {'alpha': 0.1450667028156452, 'beta': 2.0644775751342763e-06}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,236] Trial 87 finished with value: 0.04952770784950444 and parameters: {'alpha': 4.295544610864791e-09, 'beta': 9.013765953940027e-06}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,268] Trial 88 finished with value: 0.047028930029074434 and parameters: {'alpha': 0.30185000737739875, 'beta': 0.037606892901212566}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,297] Trial 89 finished with value: 0.049501756576875375 and parameters: {'alpha': 6.47474296092292e-08, 'beta': 0.010429757807728986}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,324] Trial 90 finished with value: 0.04552740768509274 and parameters: {'alpha': 0.5677276762790588, 'beta': 1.840475809157258e-09}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,362] Trial 91 finished with value: 0.04456035802073241 and parameters: {'alpha': 0.7917288794438357, 'beta': 0.0026136324067332914}. Best is trial 81 with value: 0.043982397960851925.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:18,396] Trial 92 finished with value: 0.04399424136181273 and parameters: {'alpha': 0.9950182592490293, 'beta': 0.00014285961644927076}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,424] Trial 93 finished with value: 0.04710017706182552 and parameters: {'alpha': 0.30417380378414044, 'beta': 6.513437943860882e-05}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,449] Trial 94 finished with value: 0.048034316826914225 and parameters: {'alpha': 0.17686366131444886, 'beta': 0.00014448897068112688}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,472] Trial 95 finished with value: 0.045488295333350345 and parameters: {'alpha': 0.5752733681057544, 'beta': 0.00046370057789735513}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,495] Trial 96 finished with value: 0.04878062303012993 and parameters: {'alpha': 0.08508215103635475, 'beta': 2.677070504439946e-05}. Best is trial 81 with value: 0.043982397960851925.
C:\anaconda\lib\site-packages\optuna\samplers\_tpe\parzen_estimator.py:188: RuntimeWarning: divide by zero encountered in true_divide
  coefficient = 1 / z / p_accept
[I 2022-06-19 10:28:18,519] Trial 97 finished with value: 0.044001735839986726 and parameters: {'alpha': 0.9888661011176177, 'beta': 0.0023587919639410557}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,544] Trial 98 finished with value: 0.04675159601980808 and parameters: {'alpha': 0.33101125757388394, 'beta': 0.07714710327989109}. Best is trial 81 with value: 0.043982397960851925.
[I 2022-06-19 10:28:18,565] Trial 99 finished with value: 0.049442881531157196 and parameters: {'alpha': 0.008842841989637864, 'beta': 0.0018563907470587047}. Best is trial 81 with value: 0.043982397960851925.
print(study.best_params)
{'alpha': 0.9921226833180047, 'beta': 0.007317691582675192}
y_pred1 = model1.predict(data_norm)
y_pred2 = model2.predict(data_norm[0])
y_pred3 = model3.predict(data_norm)
y_pred = (study.best_params['alpha']*y_pred1+
          study.best_params['beta']*y_pred2+
          (1-study.best_params['alpha']-study.best_params['beta'])*y_pred3)
print('RMSLE линейной регрессии:{0:.5}'.format(rmsle_err(y, y_pred1)))
print('RMSLE изотенической регрессии:{0:.5}'.format(rmsle_err(y, y_pred2)))
print('RMSLE ElasticNet:{0:.5}'.format(rmsle_err(y, y_pred3)))
print('RMSLE ансамбля:{0:.5}'.format(rmsle_err(y, y_pred)))
RMSLE линейной регрессии:0.19453
RMSLE изотенической регрессии:0.21608
RMSLE ElasticNet:0.20691
RMSLE ансамбля:0.19456
 
