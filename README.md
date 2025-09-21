# 🚌 Redbus Data Scraping & Dynamic Filtering

## 📌 Project Overview  
This project scrapes bus data from [Redbus](https://www.redbus.in/) using **Selenium**, stores it in **SQLite**, and displays it in an interactive dashboard built with **Streamlit**.  

It allows users to filter buses by:
- Bus type (Sleeper/Seater/AC/Non-AC)
- Price range
- Star rating
- Seat availability  

This makes it useful for **travel aggregators**, **market analysis**, and **competitor insights**.

---

## 🦄 Tech Stack
- **Python**
- **Selenium** (for scraping)
- **SQLite** (for storing scraped data)
- **Pandas** (for data manipulation)
- **Streamlit** (for dashboard & filtering)

raped and stored in SQLite DB successfully.")
#scrapper.py
from selenium import webdriver
from selenium.webdriver.common.by import By
import time
import sqlite3

def create_db():
    conn = sqlite3.connect("redbus.db")
    cursor = conn.cursor()
    cursor.execute('''CREATE TABLE IF NOT EXISTS bus_routes (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        route_name TEXT,
                        route_link TEXT,
                        busname TEXT,
                        bustype TEXT,
                        departing_time TEXT,
                        duration TEXT,
                        reaching_time TEXT,
                        star_rating FLOAT,
                        price DECIMAL,
                        seats_available INT)''')
    conn.commit()
    conn.close()

def scrape_redbus():
    driver = webdriver.Chrome()  # Make sure ChromeDriver is installed
    driver.get("https://www.redbus.in/")

    time.sleep(5)  # allow page to load fully

    # Example: Search for buses from Chennai to Bangalore
    from_city = driver.find_element(By.ID, "src")
    from_city.send_keys("Chennai")

    to_city = driver.find_element(By.ID, "dest")
    to_city.send_keys("Bangalore")

    date_box = driver.find_element(By.ID, "onward_cal")
    date_box.click()
    driver.find_element(By.XPATH, "//td[@class='wd day']").click()

    driver.find_element(By.ID, "search_btn").click()
    time.sleep(8)

    buses = driver.find_elements(By.XPATH, "//div[@class='clearfix bus-item-details']")

    conn = sqlite3.connect("redbus.db")
    cursor = conn.cursor()

    for bus in buses[:20]:  # scrape first 20 results
        try:
            busname = bus.find_element(By.CLASS_NAME, "travels").text
            bustype = bus.find_element(By.CLASS_NAME, "bus-type").text
            departing_time = bus.find_element(By.CLASS_NAME, "dp-time").text
            duration = bus.find_element(By.CLASS_NAME, "dur").text
            reaching_time = bus.find_element(By.CLASS_NAME, "bp-time").text
            star_rating = bus.find_element(By.CLASS_NAME, "rating").text if bus.find_elements(By.CLASS_NAME, "rating") else None
            price = bus.find_element(By.CLASS_NAME, "fare").text.replace("₹", "").strip()
            seats = bus.find_element(By.CLASS_NAME, "seat-left").text.replace("Seats left", "").strip()

            cursor.execute('''INSERT INTO bus_routes 
                            (route_name, route_link, busname, bustype, departing_time, duration, reaching_time, star_rating, price, seats_available) 
                            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)''',
                           ("Chennai-Bangalore", "https://www.redbus.in/", busname, bustype, departing_time, duration, reaching_time, star_rating, price, seats))
        except Exception as e:
            print("Error:", e)
            continue

    conn.commit()
    conn.close()
    driver.quit()

if __name__ == "__main__":
    create_db()
    scrape_redbus()
    print("Data scraped and stored in SQLite DB successfully.")
#app.py
import streamlit as st
import sqlite3
import pandas as pd

def load_data():
    conn = sqlite3.connect("redbus.db")
    df = pd.read_sql_query("SELECT * FROM bus_routes", conn)
    conn.close()
    return df

st.title("🚌 Redbus Data Explorer")

df = load_data()

# Sidebar filters
bustype = st.sidebar.multiselect("Select Bus Type", df["bustype"].unique())
price_range = st.sidebar.slider("Select Price Range", int(df["price"].min()), int(df["price"].max()), (500, 1500))
rating = st.sidebar.slider("Select Minimum Rating", 0.0, 5.0, 3.0)

# Filtering
filtered_df = df.copy()
if bustype:
    filtered_df = filtered_df[filtered_df["bustype"].isin(bustype)]
filtered_df = filtered_df[(filtered_df["price"].astype(float) >= price_range[0]) & 
                          (filtered_df["price"].astype(float) <= price_range[1])]
filtered_df = filtered_df[filtered_df["star_rating"].astype(float) >= rating]

st.dataframe(filtered_df)

st.write("### Summary Stats")
st.write(filtered_df.describe())




