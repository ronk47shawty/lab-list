[README.md](https://github.com/user-attachments/files/34703172/README.md)
# Lab List

NBA player prop research site. `build.py` pulls the latest box scores and schedule
(ESPN data via sportsdataverse) and writes `index.html`. A GitHub Action reruns it
every 2 hours, so the live site keeps itself up to date.

Run locally: `pip install pandas pyarrow pillow && python build.py`
