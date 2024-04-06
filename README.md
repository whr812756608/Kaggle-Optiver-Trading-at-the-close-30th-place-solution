# Kaggle-Optiver-trading-at-the-close-30th-place-solution

-The link to the competitions: [https://www.kaggle.com/competitions/optiver-trading-at-the-close](https://www.kaggle.com/competitions/optiver-trading-at-the-close)

-The link to our team's origin post: https://www.kaggle.com/competitions/optiver-trading-at-the-close/discussion/462650

-This is a competion with a 3 month forecast period, where all the models in this competition are running against real market data with update in every two week.

**Origin Post**
The long journey has ended. Thanks to Optiver and Kaggle for hosting this amazing competition. Thanks to my teammates @chellyfan and @unknownkid for their incredible effort. We ended up in 30th place. Here's how our rank changed during each update:

| Update             | Rank | Rank Change     |
|--------------------|------|-----------------|
| Public leaderboard | 18th | --              |
| 1st update         | 37th | ↓ 19            |
| 2nd update         | 41st | ↓ 4             |
| 3rd update         | 39th | ↑ 2             |
| 4th update         | 40th | ↓ 1             |
| Final leaderboard  | 30th | ↑ 10            |

In short , our model is an **LGBM & CatBoost ensemble** trained on a total of 480 + 37 days **without online learning** (480 days from the training set and an extra 37 days from the original public leaderboard data obtained via API). Our online learning crashed when it encountered the interim update data. Our rank is quiet stable during the updates. Check origin post below for more detail discussion. 

**TL;DR**
- Feature engineering based on public notebooks
- Catboost feature elimination (some features works very well in Catboost but very bad in LGBM, vice-versa.)
- LGBM and Catboost ensemble model, 202 features in LGBM, 141 in Catboost
- Online learning to collects hidden public leaderboard data continuously update the model.  
- Our online learning works fine with public LB data but crash once we enter the interim update date period.
- Stick with single fold holdout CV
- Try different post processing approach
- Full code here : https://www.kaggle.com/code/wuhongrui/public-5-3227-lgbm-catboost-with-online-learning/notebook

Some detail discussion below
- **Feature engineering**
  - **For LGBM:**
    - Drop all the rows that contain NaN in columns like `wap`, `bid_price`, `ask_price` from the original `train.csv`.
    - Adapt most of the features by @lblhandsome ["⚡Optiver 🚀Robust Best ⚡ Single Model"](https://www.kaggle.com/code/lblhandsome/optiver-robust-best-single-model), ensuring to group by `stock_id` and `date_id` when calculating `diff`, `pct_change` and `shift`.
    - Calculate the `index wap`like this post ["Obtaining past target values with wap"](https://www.kaggle.com/competitions/optiver-trading-at-the-close/discussion/462022) , and then compute its `diff`, `pct_change`, and `shift`.
    - Use features from the previous day at the same seconds to predict today's target
    - Further we grab the target from the previous day as a feature.
  
  - **For CatBoost:**
    - Threw all the features into CatBoost, including those that didn't work well with LGBM.
    - Utilized feature elimination in CatBoost (@yekenot ["Feature Elimination by CatBoost"](https://www.kaggle.com/code/yekenot/feature-elimination-by-catboost) to filter out less important features, 380+ -> 141.
    - Technical indicators like RSI, MACD, and Bollinger Bands work well with CatBoost but not as effectively with LGBM. But it took too long to generate them.
    - CatBoost trains significantly faster than LGBM (7 minutes vs 1 hour in holdout CV), and after adding and selecting features, our CatBoost CV/LB performance are close to our LGBM.

- **Online learning**
  - We believe this is crucial for this type of prediction competition. The aim is to collect private data and continuously retrain and update the model.
  - The incremental learning methods provided by LGBM and Catboost were not effective, leading to a lot of drifts
  - Our logic is concat all the test data in same date_id to form a cache.  Then generate features for each cache (i.e., one day) and concatenate them to the original features dataframe, which contains features generated from days 0-480.
  - The revealed targets are also concatenated to the original target dataset.
  - Retrain our model every 15 days using the concatenated data and it finally gets us to public LB 5.3231. (training at dates 481, 481+15, 481+30).
  - However once our code enter the period of interim update data, it crash and cause submission score error. We waste tons of submission try to debug but still can't locate the reason.
  - Things we have tried to debug but not work: 
    - There are missing stock in test data so the shape of each test iteration is not always (200,16), same if there are not excatly 55 time id in each days. This will lead to the shape of cache for each date_id is not always 11000 rows.(200 x 55)
    - In this case, we couldn't figure out the shape of corresponding revealed target provided by the API next day. 
 [See my post here](https://www.kaggle.com/competitions/optiver-trading-at-the-close/discussion/461989)
    - Tried merging data with `date_id`, `seconds_in_bucket`, `stock_id` instead of direct concatenation, but it didn’t resolve the issue.
    - Checked the data type for each test iteration as discussed in [[Finding] the Mechanism of Newly Added Data](https://www.kaggle.com/competitions/optiver-trading-at-the-close/discussion/456435) .
    - Considered orther edge cases like `NaN` in `date_id`, missing `seconds_in_bucket` at 0 seconds, etc.
    - Reviewed and tried solutions from all submission score error post in Discussions, but none were applicable to our case.
  
We are curious and eager to hear the way how others deal with those interim update data and utilized them to retrain the model. Debug with this API without knowing much useful information is very annoying....
