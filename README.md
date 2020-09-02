# Quevico Engineer Salary prediction 1st place solution

- 1st place (RMSE: 19449.2724)
- Competition URL: https://quevico.com/ja/competitions/105

## コンペ概要

- StackOverFlowのアンケートからアンケート回答者の給料を予測する
- 評価指標はRMSE
- アンケートの内容は自由記述はなく、大きく以下のパターンの設問・回答に分かれる
  - 選択式（１つ選ぶ）
  - 選択式（複数回答可）
  - 選択肢内で優先順位をつける（３つあれば優先度高いものから1,2,3とつけていく）

## ソリューション概要

- Model: LightGBM single (only fold ensumble)
- Validation: KFold(5fold)
- Feature Engineering
  - Target Encoding中心に約2700個
  - 詳しくは後ろに記載
- HyperParameter
  - Objective: Poisson
  - その他は軽くチューニング


## Feature Engineering

### 単独特徴量

- Multi-Category(複数選択可）
  - OneHot Encoding
  - OneHot + TruncatedSVD
  - 選択した数の合計
- Single Category
  - Label Encoding
  - Target Encoding (OutOfFoldで計算)
- 優先順位
  - そのままの数値
  - 選択肢のペアの相対順位
    - 上位を優先するため、aとb(a < b)だったとすると (b - a) / log(2+a) として計算
    - Recommendの評価指標で使われるnDCGの式を流用

### カラム間の相互特徴量

- Aggregation
  - Category(Single + MultiのOne-hot）をキー、数値(優先順位 + MultiのOne-hot)を値として以下のAggregationFeatureを作成
    - mean, std, max, min
    - meanと該当行の数値の差、比率
  - パターンが多すぎるため、キーも数値もシンプルなモデルでのFeatureImportance上位50件に含まれるものに絞った
- Multi-TargetEncoding
  - カテゴリ(Single + MultiのOne-hot)のペアを一つのカテゴリとして、TargetEncodingを実施
  - カテゴリの相互作用を特徴化するのが目的
  - Aggregation同様組み合わせが多いため、上位30件の特徴のみを対象とした


## Not Work

- Feature
  - Triple-TargetEncoding
  - Multi-CountEncoding
  - Multi-Category+他カテゴリーでのLDA, NMF, SVD
  - Target EncodingのSmoothing
  - 欠損値を予測
    - 重要度の高いカラムで欠損しているものを他のデータからLightGBMで学習して予測して埋めた
- Model
  - Catboost
  - NeuralNet
- Parameter
  - Objective: huber, tweedie, fair
- Data
  - ノイズ対策
    - 明らかに給料が高すぎる or 働いているのに給料が低すぎる人がいた
    - OOF predictionの誤差が大きすぎるデータをノイズデータとして取り除いた上で際学習
  - 分布によるデータわけ
    - 普通の給料をもらっている分布と明らかに給料が少ない二つの分布に分かれていた
    - 給料のタイプがWeeklyとMonthlyもしくは職業が学生といった条件のユーザを抽出すると後者のユーザと分けられた
    - それを活用して以下を実施したが全部ダメだった
      - 二つに分けてのモデル学習(Objectiveをそれぞれ最適化してもダメだった）
      - 一つのモデルで学習したあと、斬差をそれぞれのモデルで学習
- Other
  - Binに区切ったStratifiedKFold
    - 精度は変わらなかったが、最初からこっちにしてたほうがValidationが安定してた可能性はある
 
