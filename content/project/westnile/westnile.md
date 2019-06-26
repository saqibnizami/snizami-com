
+++
title = "Project 4: West Nile Prevention Analysis"
date = "2018-06-15"
author = "Saqib Nizami"
categories = "Projects"
+++



![Project 4](p4title.png)

_Summary : This was a group project that presented many opportunities to engineer features and try out different visualizations and models. [Bokeh](https://bokeh.pydata.org/en/latest/) was used for visualizing map data, and the [XGBoost](https://xgboost.readthedocs.io/) library was used for the models. There were additional considerations to be made due to the unbalanced classes, geospatial data, and missing data._

---

### Objective:

Help the City of Chicago combat West Nile Virus spread by analyzing weather, previous spraying efforts, and mosquito trap data.

---
West Nile Virus (WNV) is spread by mosquitos and can cause fever, encephalitis, meningitis, or death. It has been present in the United States since 1999 and was reported in Chicago in 2002. Our focus in this project is to use our predictions to make effective recommendations to combat WNV.

### Exploratory Data Analysis
The locations of previous sprays, weather stations, and West Nile positive mosquito traps, as well as the species of mosquito found in the traps is mapped below in a Bokeh + Google Maps visualization. The size of the mosquito species circle denotes the number in the trap.


```python
output_notebook()
map_options = GMapOptions(lat=41.8781, lng=-87.6298, map_type="roadmap", zoom=11, styles=mapstyle)
tools = 'pan,wheel_zoom,zoom_in,zoom_out,save,reset'

p = gmap("AIzaSyCB1nfQBErvWg4B36laI1EtdzGRWwMo2wU", map_options, title="Chicago", 
         tools=tools, width=1280, height=720)

pos_trap = ColumnDataSource(wnvpresentmap[['Longitude','Latitude','Trap']])
p.circle(x="Longitude", y="Latitude", 
         size=2, name='traps', source=source, 
         fill_color='palegreen', fill_alpha=0.05, 
         line_color='palegreen', line_alpha=0.05, legend="Postive Trap Locations")

weathermap = ColumnDataSource(stations) 
p.triangle(x="Longitude", y="Latitude", 
           size=10,name="weather_stations",
           fill_color='orangered', fill_alpha=0.01, 
           line_color='orangered', line_alpha=0.001,
           source=(weathermap), legend='Weather Station')

spraymap = ColumnDataSource(spray)
p.square(x="Longitude", y="Latitude", 
         name='spray', source=spraymap, 
         fill_color='mistyrose', fill_alpha=0.05, 
         line_color='mistyrose', line_alpha=0.05, legend='Spray Route')



# Species
species_cm = {'CULEX ERRATICUS' : 'green',
              'CULEX PIPIENS': 'yellow',
              'CULEX PIPIENS/RESTUANS': 'magenta', 
              'CULEX RESTUANS': 'cyan', 
              'CULEX SALINARIUS':'blue', 
              'CULEX TARSALIS':'gray', 
              'CULEX TERRITANS': 'grey'}

colors = [species_cm[x] for x in speciesmap['Species']]
sizes = [(len(speciesmap[speciesmap["Species"]==x])/len(speciesmap["Species"]))*200 for x in speciesmap.Species.unique()]
species = {'lon': speciesmap['Longitude'],
           'lat': speciesmap['Latitude'],
           'species': speciesmap['Species'],
           'color': colors,
           'size': sizes}
p.circle('lon', 'lat',source=species, size='size', color='color', fill_alpha=0.1, line_alpha=0.5, legend='species')

show(p)
```



    <div class="bk-root">
        <a href="https://bokeh.pydata.org" target="_blank" class="bk-logo bk-logo-small bk-logo-notebook"></a>
        <span id="79275506-393c-4394-a277-3566c4196667">Loading BokehJS ...</span>
    </div>




    /Users/saqibnizami/anaconda3/envs/dsi/lib/python3.6/site-packages/bokeh/models/sources.py:114: BokehUserWarning: ColumnDataSource's columns must be of the same length. Current lengths: ('color', 10506), ('lat', 10506), ('lon', 10506), ('size', 7), ('species', 10506)
      "Current lengths: %s" % ", ".join(sorted(str((k, len(v))) for k, v in data.items())), BokehUserWarning))




<div class="bk-root">
    <div class="bk-plotdiv" id="68e10ee2-5ab2-46ac-80c6-65e120f05bde"></div>
</div>





#### Feature Engineering

We did a considerable amount of clean-up and feature engineering. Using the weather, spray, and trap data:
1.  rolling averages were calculated for six intervals from the preceding 3-30 days.
2. heavy and light rain days were identified - as rain is conducive to mosquito hatching but heavy rain can disrupt and wash away eggs.
3. relative humidity, calculated by temperature and dew-point features and using a formula from the National Oceanic and Atmospheric Administration (NOAA):

```python
def humidity(Tf, Tdf):

    # convert temp to celsius
    Tc=5.0/9.0*(Tf-32.0)
    # convert dewpoint temp to celsius
    Tdc=5.0/9.0*(Tdf-32.0)
    # saturation vapor pressure
    Es=6.11*10.0**(7.5*Tc/(237.7+Tc))
    # actual vapor pressure
    E=6.11*10.0**(7.5*Tdc/(237.7+Tdc))
    #relative humidity
    RH =(E/Es)*100

    return RH

train_clean["RelHum"] = humidity(train_clean['Tavg'],train_clean['DewPoint'])
test_clean["RelHum"] = humidity(test_clean['Tavg'],test_clean['DewPoint'])
```

4. Time since WNV was present on the block

### Modeling: XGBoost

XGBoost stands for "eXtreme Gradient Boost", and was developed by [Tianqi Chen](https://arxiv.org/abs/1603.02754). It builds upon [Gradient Boosting](https://en.wikipedia.org/wiki/Gradient_boosting), which itself is a form of [Boosting](https://en.wikipedia.org/wiki/Boosting_(machine_learning).
As a very high level overview, __Boosting__ is an ensemble technique where the model progresses by fixing the errors of the previous iteration. This is a sequential process that occurs until no more improvements can be made. __Gradient Boosting__ is a technique where the next iteration of the model predicts the errors of the previous one, all of which are added together to make a final prediction. It uses gradient descent to minimize loss when adding new models. In this instance, both of these are applied in the context of Decision Trees - where each split is conducted using Gradient Boosting.

XGBoost has some advantages over normal gradient boosting, which led to my interest in the algorithm:

1. Regularization: XGBoost incorporates regularization as into its tree creation as a measure against overfitting.
2. Parallel Processing: Boosting is a sequential process (each node is dependent on the results of the previous split), but XGBoost incorporates parallel processing - making it faster than regular gradient boosting.
3. Customization: XGBoost allows for the definition of custom optimization objectives and evaluation criteria.
4. Missing Values: A parameter for missing values can be passed. When the algorithm encounters a missing value, it will incorporate that parameter into the values it tests it on each node. It will then try to learn what path to take when dealing with future missing values.
5. Tree Pruning: XGBoost splits upto the specified `max_depth` and then prunes backwards, removing splits where there was no positive gain.
6. The algorithm offers built in cross-validation. The user can run a cross-validation at every iteration of the boosting process and thus get the optimum number of boosting iterations in a single run.
7. Continuable: A XGBoost model can be restarted from the last iteration of its previous run.

However, as fast and feature-rich as XGBoost sounds, tuning the parameters was a involved yet rewarding endeavor - as I will demonstrate in the classification model section.

#### XGBoostRegressor Model: Missing Mosquito Number Imputation

Some of the data was missing Mosquito Numbers, therefore to predict incidence of WNV and assess spray effectiveness, our group decided to impute the values with a regression model.
A XGBoostRegressor model predicted number of mosquitos with 0.73 accuracy. The model was fit with data from 2007 and 2009, which did not have spray data, and then made to predict number of mosquitos if there was no spraying in 2011 and 2013. Below is a regression plot showing the actual number of mosquitos (in 2011 and 2013) vs our model's predictions for those years.

![Predicted Number of Mosquitos](num_mos_xgbreg.png)

Using the regression model to impute values allowed us to conduct a spray effectiveness analysis for 2011 and 2013:

![Spray Effectiveness](sprayeff2.png)
![Spray Effectiveness table](sprayeff.png)

We can see there is a considerable population reduction with spraying. The drop-off in effectiveness between short term and long term is also minimal.

#### XGBoostClassifier model

XGBoost offers a large amount of flexibility when it comes to hyper-parameter tuning. This flexibility can also be incredibly daunting though, so I took a generalized initial approach to tuning for XGBoost:
1. Use a high learning rate, somewhere between 0.05 and 0.3, then use the built in cross-validation to determine the optimum number of trees needed.
2. Tune the tree parameters: `max_depth`, `min_child_weight`, `gamma`, `subsample`, `colsample_bytree` for the learning rate and tree number.
3. Tune the regularization parameters: `lambda` and `alpha` to reduce model complexity and stave off over-fitting.
4. Lower the learning rate to decide the optimal parameters.

For each step of tuning, the parameter choices were `GridSearch`ed over to find the optimal settings. The final model settings are below:

```python
xgb5 = XGBClassifier(learning_rate =0.01,
                     n_estimators=621,
                     max_depth=2,
                     min_child_weight=3,
                     gamma=0,
                     subsample=0.975,
                     colsample_bytree=0.95,
                     reg_alpha = 100,
                     objective= 'binary:logistic',
                     nthread=4,
                     scale_pos_weight=1, seed=27)

modelfit(xgb5, train_clean, predictors)

    ```
    Model Report
    Accuracy : 0.9467
    AUC Score (Train): 0.808906

#### Feature Extraction
The model was used to extract the 50 most important features in the dataset for our purposes:

![top50](TopFeaturesLatest.png)

Most of the top features chosen by the model were our engineered weather features.

#### Imbalanced Classes and pipeline
Since there is the problem of imbalanced classes in this dataset ( Non-presence of WNV is much more frequent than the presence of WNV), I used a `BalancedBaggingClassifier` from the `imbalanced-learn` package in the final model pipeline.

The pipeline incorporated a `StandardScaler` then a `BalancedBaggingClassifier` that used `GridSearchCV` for the parameters and our XGBoostClassifier model as its base estimator.

                    precision    recall  f1-score
        avg / total       0.90      0.95      0.92

The analysis could benefit from a more complete dataset, therefore more spraying would help cut down the noise in the data. To demonstrate the difficulty in gaining meaningful information from this data, a Radial Visualization from the `yellowbrick` library was used. This is a multivariate data visualization that plots the features along the circumference of the circle and the data points for each class radiating out from the center towards the features. Ideally, one would like to see the classes (in this case, 0 = WNV Not Present, and 1 = WNV Present) to not have too much overlap and graviate towards different features. As we can see below, there is considerable overlap between the classes in this problem, reinforcing the idea that drawing very confident conclusions from the current dataset is difficult.

![radviz](radviz.png)

### Cost-Benefit analysis

West Nile Virus has cost the U.S. ~\\(800M in healthcare and lost productivity since 1999. The average cost is \\)20,000 per case. Chicago reports ~20-40 cases annually – a total implied cost of ~\\(400-800K. Spraying would cost ~\\)350-500K annually (assuming 50-75% of areas are sprayed) and cause a 75% reduction in number of mosquitoes (benchmarks – 90%)



