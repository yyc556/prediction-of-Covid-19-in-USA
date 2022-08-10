# Covid-19在美國之確診數預測
<br>深度學習概論-英家慶 課程期末專題
<br>組員：陳彥妤、鄭巧翎、黃翊瑄
* 資料集
<br>[Novel Corona Virus 2019 Dataset | Kaggle](https://www.kaggle.com/datasets/sudalairajkumar/novel-corona-virus-2019-dataset)
* 使用PyTorch
* 模型建立
<br>Simple RNN, LSTM, GRU, CNN+LSTM

## 資料預處理
1. 累加確診人數
```
import pandas as pd
time_series_data = pd.DataFrame()
df = pd.DataFrame(df[df.columns[11:]].sum(), columns=['confirmed'])
time_series_data = pd.concat([time_series_data,df], axis=1)
time_series_data.index = pd.to_datetime(time_series_data.index, format='%m/%d/%y')
time_series_data.reset_index(inplace=True)
```
<img src="https://github.com/yyc556/prediction-of-Covid-19-in-USA/blob/main/images/cumulative%20confirmed%20cases.png">

2. MinMaxScaler
利用MinMaxScaler將數據縮到[0,1]之間
```
from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
scaler = scaler.fit(np.expand_dims(time_series_data['confirmed'], axis=1))

time_series_data['confirmed'] = scaler.transform(np.expand_dims(time_series_data['confirmed'], axis=1))
```
3. 切割資料集
```
split = round(0.8*len(time_series_data))
train_data = time_series_data['confirmed'][:split]
test_data = time_series_data['confirmed'][split:]
```
4. 建立序列
利用前五天的數據預測第六天之確診人數
```
def create_sequences(data, previous):
  X, y = [], []
  for i in range(len(data)-previous-1):
      x = data[i:(i+previous)]
      X.append(x)
      y.append(data[i+previous])
  X = np.array(X)
  y = np.array(y)

  return X, y

previous = 5  # 利用前五天預測第六天的確診人數
X_train, y_train = create_sequences(train_data.to_numpy(), previous)
X_test, y_test = create_sequences(test_data.to_numpy(), previous)
X_train = torch.from_numpy(X_train).float()
X_test = torch.from_numpy(X_test).float()
y_train = torch.from_numpy(y_train).float()
y_test = torch.from_numpy(y_test).float()
```
5. 統一資料維度
```
X_train = X_train.unsqueeze(2)
X_test = X_test.unsqueeze(2)
y_train = y_train.unsqueeze(1)
y_test = y_test.unsqueeze(1)
```

## 模型建立
* Simple RNN
```
class RNN(nn.Module):
  def __init__(self, input_size, hidden_size, num_layers, num_class):
    super(RNN, self).__init__()
    self.hidden_size = hidden_size
    self.num_layers = num_layers
    self.rnn = nn.RNN(input_size, hidden_size, num_layers, batch_first=True)
    self.fc = nn.Linear(hidden_size, num_class)
    
  def forward(self, x):
    out, _ = self.rnn(x)
    out = out[:, -1, :]
    out = self.fc(out)
    return out
```
* LSTM
```

class LSTM(nn.Module):
  def __init__(self, input_size, hidden_size, num_layers, num_class):
    super(LSTM, self).__init__()
    self.hidden_size = hidden_size
    self.num_layers = num_layers
    self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
    self.fc = nn.Linear(hidden_size, num_class)
    
  def forward(self, x):
    h0 = Variable(torch.zeros(num_layers, x.size(0), hidden_size))
    c0 = Variable(torch.zeros(num_layers, x.size(0), hidden_size))
    out, _ = self.lstm(x, (h0, c0))
    out = out[:, -1, :]
    out = self.fc(out)
    return out
```
* GRU
```
class GRU(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers, num_class):
        super(GRU, self).__init__()
        self.num_layers = num_layers
        self.hidden_size = hidden_size

        self.gru = nn.GRU(input_size, hidden_size, num_layers, batch_first=True)
        self.linear = nn.Linear(hidden_size, num_class)
        
    def forward(self, x):
        out, _ = self.gru(x)
        out = out[:, -1, :]
        out = self.linear(out)
        return out
```
* CNN+LSTM
```
class ConvLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers, num_class):
        super(ConvLSTM, self).__init__()
        self.num_layers = num_layers
        self.hidden_size = hidden_size
        
        self.conv = nn.Sequential(
            nn.Conv1d(5, hidden_size, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(stride=1, kernel_size=1)
        )
        self.LSTM1 = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        self.LSTM2 = nn.LSTM(hidden_size, hidden_size, num_layers, batch_first=True)
        self.linear = nn.Linear(hidden_size, num_class)

    def forward(self, x):
        out = self.conv(x)
        out, _ = self.LSTM1(out)
        out, _ = self.LSTM2(out)
        out = out[:, -1, :]
        out = self.linear(out)
        return out
```
模型架構整理
<img src="https://github.com/yyc556/prediction-of-Covid-19-in-USA/blob/main/images/model%20structure.png">

## 模型訓練
<br>Loss Function：MSE
<br>Optimizer：Adam
<br>Learning Rate：0.001
<br>Epoch：200
```
def train_model(model, X_train, y_train, X_test=None, y_test=None):
  loss_fn = nn.MSELoss()
  optimizer = opt.Adam(model.parameters(), lr = 0.001)
  num_epoches = 200

  train_loss_hist = np.zeros(num_epoches)
  test_loss_hist = np.zeros(num_epoches)

  for epoch in range(num_epoches):
    optimizer.zero_grad()
    y_pred = model(X_train)
    loss = loss_fn(y_pred, y_train)
    loss.backward()
    optimizer.step()
    if X_test is not None:
      with torch.no_grad():
        y_test_pred = model(X_test)
        test_loss = loss_fn(y_test_pred, y_test)
      test_loss_hist[epoch] = test_loss.data

      if (epoch+1)%10 == 0:
        print('Epoch: %d, Train Loss: %.4f, Test Loss: %.4f' %  (epoch+1, loss.data, test_loss.data))
    else:
      if (epoch+1)%10 == 0:
        print('Epoch: %d, Loss: %.4f' %  (epoch+1, loss.data))
    train_loss_hist[epoch] = loss.data 
    
  return y_pred, train_loss_hist, test_loss_hist  
```


