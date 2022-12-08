# Clash Royale Analysis 

> This content is not affiliated with, endorsed, sponsored, or specifically
> approved by Supercell and Supercell is not responsible for it. For more
> information see Supercell’s Fan Content Policy:
> www.supercell.com/fan-content-policy.

An exercise in data collection/analysis, juggling with millions of records.

<div align="center"><p>
    <a href="https://github.com/S1M0N38/cr-analysis/pulse">
      <img alt="Last commit" src="https://img.shields.io/github/last-commit/S1M0N38/cr-analysis?style=for-the-badge&color=8bd5ca"/>
    </a>
    <a href="https://github.com/S1M0N38/cr-analysis/blob/main/LICENSE">
      <img alt="License" src="https://img.shields.io/github/license/S1M0N38/cr-analysis?style=for-the-badge&color=ee999f" />
    </a>
    <a href="https://github.com/S1M0N38/cr-analysis">
      <img alt="Repo Size" src="https://img.shields.io/github/repo-size/S1M0N38/cr-analysis?color=DDB6F2&label=SIZE&style=for-the-badge" />
    </a>
    <a href="https://discord.com/users/S1M0N38#0317">
      <img alt="Discord" src="https://img.shields.io/static/v1?label=DISCORD&message=DM&color=a3b7ff&style=for-the-badge" />
    </a>
</div>

-------------------------------------------------------------------------------

## Data Collection

First we need data. Clash Royale data can be retrieved using [Clash Royale
API](https://developer.clashroyale.com/#/). The date we are interested in are
battles between players, specifically 1v1 top players battles.

### How Data Collection works

Clash Royale base its match making algorithm on players strength, i.e. you
battle against other players that are more or less strong as you are. So, to
collect battles played between top players, you can start requesting battles
from a bunch of top players (we call them `root_players`). Root players'
opponents are also top players so you can request also their battles and the
process goes on.

So to collect data you have to constantly send http requests to the API and use
response to produce other http calls. `data/collect.py` is a python script
designed to do exactly that: request battles, save battles into a csv file,
generate new requests based on collect battles.

`data/collect.py` make use of `data/crawler.py` which implement an asynchronous
iterator class capable of scheduling parallel http requests based on previous
API response. 

Data are stored in csv compressed files (suffix `.csv.gz`). We choose this
format because it's easy to work with and widely used/supported.


### How Data Collection is performed

At the time of writing we use Raspberry Pi 3 Model B
([configuration](https://github.com/S1M0N38/dots/tree/rpi)) to collect data
running `data/collect.sh`. `data/collect.sh` is a bash script that wrap
`data/collect.py` specifying command line arguments (e.g. root players, number
of parallel requests, name of the output file, etc.). `data/collect.sh` is
scheduled to run hourly by [crontab](https://en.wikipedia.org/wiki/Cron) and it
takes about 50 minutes to complete on Raspberry Pi with our connection. Moreover
`data/collect.sh` is responsible for sorting and compressing csv file that
`data/collect.py` has produced. Data generated by `data/collect.py` are stored
in `db/hours` folder.

`data/join.sh` is bash script that is run daily to join data from the previous
day eliminating redundant data. Results of `data/join.sh` are stored in
`db/days`.


### Set up Data Collection

Here's how to setup data collection pipeline on your own machine. Assuming you
have access to command line and [git](https://git-scm.com/) installed ... 

1. Clone this repo in your $HOME directory
```bash 
git clone https://github.com/S1M0N38/cr-analysis.git $HOME/cr-analysis
```

2. Navigate to `data` subdirectory
```bash
cd $HOME/cr-analysis/data
```
Every **data collection** script MUST be run from this directory
(`cr-analysis/data`).

3. Ensure `python 3.9` or greater is available.
``` bash
python -V
```
This project is developed using python 3.9 hence backward compatibility is not
ensured.

4. Upgrade `pip` and install requirements
```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

5. Create a developer account [here](https://developer.clashroyale.com/#/) and
   add credentials as environment variables. For example if you use bash as
   shell add
```bash
export API_CLASH_ROYALE_EMAIL="example@mail.com"
export API_CLASH_ROYALE_PASSWORD="MY_P4S5W0RD"
```
to your `.bashrc`. Then restart the terminal and navigate again to
`cr-analysis/data`.

6. Run test script to ensure that data collection scripts will runs flawlessly.
```bash
./test.sh
```

7. If not error arose you can schedule `collect.sh` to run hourly and `join.sh`
   to run daily.
```bash 
crontab -e
```
This command will open a text file with your favorite editor (choose `nano` if
in doubt). Every line in this file is a scheduled task. Lines that starts with
`#` are just comments. Add those lines at the end of this file.
```crontab
# Clash Royale API credentials
# (you have to add credentials to this file as well)
API_CLASH_ROYALE_EMAIL="example@mail.com"
API_CLASH_ROYALE_PASSWORD="MY_P4S5W0RD"

  0  *  *  *  *  cd $HOME/cr-analysis/data && ./collect.sh 100000
  0 12  *  *  *  cd $HOME/cr-analysis/data && ./join.sh
# |  |  |  |  |  |
# |  |  |  |  |  +--- command
# |  |  |  |  +----- day of week (0 - 6) (Sunday=0)
# |  |  |  +------- month (1 - 12)
# |  |  +--------- day of month (1 - 31)
# |  +----------- hour (0 - 23)
# +------------- min (0 - 59)
```
That's should be all. Scripts scheduled with `crontab` will run in background.
Use a process viewer (e.g. [htop](https://htop.dev/)) to check if the are
running. Alternatively take a look at the log files contained in `db/hours` and
in `db/days`.

### Further Data Manipulation

Let's pretend that tree day pass since you have collection pipeline running on
your machine. You should have a directory structure like this
```
cr-analysis
├── db
│  ├── days
│  │  ├── 20221114.csv.gz
│  │  ├── 20221115.csv.gz
│  │  └── 20221116.csv.gz
│  ├── hours
│  │  └── ...
│  ├── test
│  │  └── ...
│  └── ...
├── ...
└── README.md
```
In `cr-analysis/db/days` are stored compressed CSV files containing battles
collected on that date (This does not mean that battles were played on that day
but that they were played on that day or days before). Now you can copy
`db/days` from your "scraping machine" (in our case Raspberry Pi) to your
"analysis machine" (in our case a laptop).

Clone this very repo in your "analysis machine" and navigate to
`cr-analysis/db`. Here you can find some empty directory and some script for
data manipulation. Those empty folder will be populated with data coming from
"scraping machine".

We can connect to Raspberry with ssh so we can pull data using `rsync` from
"analysis machine":
```bash
rsync -aPv pi@192.168.178.101:cr-analysis/db/days .
```
where:
- `pi` is the user on RPI
- `192.168.178.101` is the local IP of RPI
- `cr-analysis/db/days` is the path to data folder on RPI
- `.` is where I want to save data on "analysis machine"

With CSV on our "analysis machine" we can proceed with further data
manipulation. For example we like to rearrange battles in such a way that CSV
file name refers to the days battles were played instead of the day battles data
were collected. This can be achieved using `season.sh` script. For example
```bash
./season.sh 20221107 20221205
```
will create the folder `db/20221107-20221205` with one csv file per day between
the 7th of Nov 2022 to the 5th of Dec 2022.
```
cr-analysis
├── db 
│  ├── 20221107-20221205
│  │  ├── 20221107.csv.gz
│  │  ├── 20221108.csv.gz
│  │  ├── ...
│  │  ├── 20221204.csv.gz
│  │  └── 20221205.csv.gz
│  ├── days
│  │  └── ...
│  ├── hours
│  │  └── ...
│  ├── test
│  │  └── ...
│  └── ...
├── ...
└── README.md
```
This means that the file `db/20221107.csv.gz` will contains battle played on 7th
of Nov 2022 sorted by time. This way of storing data have a couple of
advantages:

- Keep files size relatively small, so they are easy to handle compared to a
  huge CSV file.
- `gzip` compression allow to concatenate them without decompression (e.g. `cat
  20221107.csv.gz 20221108.csv.gz 20221109.csv.gz` will stream to stdin all
  battle played between 20221107 and 20221109 sorted by time).

Another way to organize data is to store them in a proper database. The script
`db/sqlite.sh` takes file .csv.gz from `db/days` and concatenate them in a
single `db/db.sqlite` file.
```
cr-analysis
├── db
│  ├── days
│  │  └── ... 
│  └── db.sqlite
└── ...
```

For files manipulation `collect/join.sh`, `db/season.sh` and `db/sqlite.sh`
leverage the power of command line programs pre-installed on many Unix-Like OS.
Take a look at them if you want to manipulate compressed CSV on your own.

-------------------------------------------------------------------------------

## Data Analysis

Data Analysis is still in early stage but here is a simple example if you'd
like to experimenting on your own. This example use
[pandas](https://pandas.pydata.org/) as data analysis tool to analyse battles
played in different days.

### Install

Assuming you have already clone this repository ...

1. Navigate to `analysis` subdirectory
```bash
cd $HOME/cr-analysis/analysis
```
Every **data analysis** command/script MUST be run from this directory
(`cr-analysis/analysis`).

2. Ensure `python 3.9` or greater is available.
``` bash
python -V
```
This project is developed using python 3.9 hence backward compatibility is not
ensured.

3. Upgrade `pip` and install requirements
```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```
This install data analysis dependencies.

### Preprocessing

First it's better to convert data files from well-known csv format to the
less-known *parquet* format on which we will have a fine grained control while
working with pandas ([this video](https://youtu.be/9LYYOdIwQXg) explains why).
Suppose we obtained `db/20221107-20221205/20221107.csv.gz` from the data
collection pipeline
```
cr-analysis
├── db 
│  ├── 20221107-20221205
│  │  ├── 20221107.csv.gz
│  │  └── ...
│  └── ...
└── ...
```
You can convert if to parquet format by using `analysis/parquet.py`
```bash
python parquet.py -i ../db/20221107-20221205/20221107.csv.gz
```
This will create `20221107.parquet` in the same directory of the input file
```
cr-analysis
├── db 
│  ├── 20221107-20221205
│  │  ├── 20221107.csv.gz
│  │  ├── 20221107.parquet
│  │  └── ...
│  └── ...
└── ...
```
The bash script `analysis/parquet.sh` convert all csv file stored in db into
parquet files.

### Simple Example

1. Start jupyerlab server with `jupyer-lab`

2. Take a look at `analysis/analysis.ipynb`
