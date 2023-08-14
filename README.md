# gStore Logigal Plan Optimizer Demonstration

## Prepare Dependecies

1. Install Python dependencies:

```bash
pip3 install -r requirements.txt
```

2. Install Node.js and yarn:

```bash
sudo apt install nodejs
sudo apt install npm
sudo npm install -g yarn
```

3. Install frontend dependencies:

```bash
cd plan
yarn install
```

4. Build frontend:

```bash
yarn build
```

5. Install gStore:

```bash
git clone https://github.com/SoftlySpoken/gStore-UO-opt.git -b demo
cd gStore-UO-opt
make pre -j
make -j
```


## Usage

1. Initialize your dataset in gStore (use `lubm` as example):

```bash
cd gStore-UO-opt
./bin/gbuild -db lubm -f ./data/lubm/lubm.nt
```

2. Start ghttp server in gStore (the port and database name should be the same as in `app.py`)):

```bash
cd gStore-UO-opt
./bin/ghttp -db lubm -p 5000
```

3. Start the backend server:

```bash
python3 app.py
```

## gStore Connection Config

Please modify the `gStore_config` variable to connect to your gStore database.

