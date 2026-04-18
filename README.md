# RKLB-Stock-Ticker
Rocket Lab Stock Ticker based off pwnagotchi hardware
# RKLB Rocket Lab Stock Ticker on Waveshare 2.13" e-Paper HAT

A low-power Raspberry Pi Zero 2 W project that displays the current **Rocket Lab (RKLB)** stock price with percentage change.  
It updates every **3 minutes** during regular market hours and every **15 minutes** during after-hours / pre-market / overnight (so you can see price movement outside trading hours).  
Time is shown in **Central Time (12-hour AM/PM)**.

The display runs quietly with very low power consumption and starts automatically on boot.

![RKLB e-Ink Ticker] https://i.imgur.com/rO408mU.jpeg

## Hardware Used

- **Raspberry Pi Zero 2 W**
- **Waveshare 2.13inch E-Ink Display HAT V4**
- 5pcs RG178 UFL SMA Coaxial Cable + connectors (for antenna if needed)
- DHT Electronics 2PCS IPEX U.FL SMD SMT Solder for PCB Mount Socket Jack Female RF Coaxial Connector
- Herfair USB to Micro USB Cable 6 Inch Right Angle
- Pwnagotchi Super 3D printed case (unmodified)

## Features

- Real-time RKLB price + percentage change (supports after-hours & pre-market)
- Big, easy-to-read price as the main focus
- Updates in **Central Time** (12-hour format with AM/PM)
- Auto-starts on boot via systemd
- Very low power usage (e-ink only refreshes when needed)

## Installation Instructions

1. Prepare the Raspberry Pi
- Install **Raspberry Pi OS Lite (64-bit)**
- Enable SSH and SPI:
  ```bash
  sudo raspi-config

2. Install Dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install git python3-pip python3-pil python3-numpy -y
sudo pip3 install yfinance --break-system-packages
3. Download Waveshare Driver

cd ~
git clone https://github.com/waveshare/e-Paper.git




4. Download the Project Files


cd ~/e-Paper/RaspberryPi_JetsonNano/python/examples
# Copy the rklb_stock_final.py file here (or git clone your repo)



5. Create the Script (rklb_stock_final.py)
cat > rklb_stock_final.py << 'EOF'
#!/usr/bin/python
import sys
import os
picdir = os.path.join(os.path.dirname(os.path.dirname(os.path.realpath(__file__))), 'pic')
libdir = os.path.join(os.path.dirname(os.path.dirname(os.path.realpath(__file__))), 'lib')
if os.path.exists(libdir):
    sys.path.append(libdir)

import logging
from waveshare_epd import epd2in13b_V4
import time
from PIL import Image, ImageDraw, ImageFont
import datetime
from zoneinfo import ZoneInfo
import yfinance as yf

logging.basicConfig(level=logging.INFO)

def get_stock_data():
    try:
        ticker = yf.Ticker("RKLB")
        df = ticker.history(period="2d", interval="5m", prepost=True)
        
        if df.empty:
            price = ticker.fast_info.get('lastPrice') or ticker.info.get('currentPrice')
            prev_close = ticker.fast_info.get('previousClose')
        else:
            latest = df.iloc[-1]
            price = round(float(latest['Close']), 2)
            regular_df = df[df.index.hour < 16]
            prev_close = regular_df.iloc[-1]['Close'] if not regular_df.empty else None
        
        if price is None:
            return None, None
        
        price = round(float(price), 2)
        
        if prev_close:
            change_pct = ((price - float(prev_close)) / float(prev_close)) * 100
            change_str = f"{change_pct:+.2f}%"
        else:
            change_str = "N/A"
        
        return price, change_str
    except Exception as e:
        logging.error(f"Data error: {e}")
        return None, None

def get_update_interval():
    now = datetime.datetime.now(ZoneInfo("America/New_York"))
    weekday = now.weekday()
    
    if weekday < 5:  # Monday to Friday
        hour = now.hour
        if 9 <= hour < 16:                    # During regular market (9:30-16:00 ET)
            return 180                        # every 3 minutes
        else:
            return 900                        # every 15 minutes outside regular hours
    else:
        return 1800                           # 30 minutes on weekends

try:
    logging.info("=== RKLB Rocket Lab Stock Ticker Started (Extended Hours + CT Time) ===")
    epd = epd2in13b_V4.EPD()
    
    font_big_price = ImageFont.truetype(os.path.join(picdir, 'Font.ttc'), 32)
    font_top       = ImageFont.truetype(os.path.join(picdir, 'Font.ttc'), 20)
    font_change    = ImageFont.truetype(os.path.join(picdir, 'Font.ttc'), 22)
    font_bottom    = ImageFont.truetype(os.path.join(picdir, 'Font.ttc'), 18)

    # Initial clear
    epd.init()
    epd.Clear()
    time.sleep(2)
    epd.sleep()

    while True:
        interval = get_update_interval()
        price, change_str = get_stock_data()
        price_str = f"${price:.2f}" if price is not None else "N/A"
        
        # Central Time in 12-hour format with AM/PM
        now_ct = datetime.datetime.now(ZoneInfo("America/Chicago"))
        time_ct = now_ct.strftime("%I:%M %p CT").lstrip("0")   # removes leading zero (e.g. 9:05 instead of 09:05)

        logging.info(f"RKLB {price_str} {change_str} | {time_ct}")

        epd.init()
        image = Image.new('1', (epd.height, epd.width), 255)
        draw = ImageDraw.Draw(image)

        # Top line
        draw.text((8, 5), "ROCKET LAB $RKLB", font=font_top, fill=0)

        # Big centered price
        draw.text((22, 37), price_str, font=font_big_price, fill=0)

        # Percentage change centered under price
        draw.text((45, 72), change_str, font=font_change, fill=0)

        # Bottom: Updated time in Central Time (12-hour AM/PM)
        draw.text((12, 98), f"Updated: {time_ct}", font=font_bottom, fill=0)

        # Display
        buffer = epd.getbuffer(image)
        epd.display(buffer, buffer)

        time.sleep(2)
        epd.sleep()

        time.sleep(interval)

except KeyboardInterrupt:
    logging.info("Stopped by user")
    try:
        epd2in13b_V4.epdconfig.module_exit()
    except:
        pass
except Exception as e:
    logging.error(f"Error: {e}")
EOF

6. Set Up Auto-Start on Boot

sudo nano /etc/systemd/system/rklb-ticker.service
Paste this service file:
[Unit]
Description=RKLB Rocket Lab Stock Ticker
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/pi/e-Paper/RaspberryPi_JetsonNano/python/examples
ExecStart=/usr/bin/python3 /home/pi/e-Paper/RaspberryPi_JetsonNano/python/examples/rklb_stock_final.py
Restart=always
RestartSec=10
StandardOutput=append:/var/log/rklb-ticker.log
StandardError=append:/var/log/rklb-ticker.log

[Install]
WantedBy=multi-user.target
Enable it:
sudo systemctl daemon-reload
sudo systemctl enable rklb-ticker.service
sudo systemctl start rklb-ticker.service


Check status:
sudo systemctl status rklb-ticker.service
7. Optional: Flip Display 180°
If the text appears upside down in your case, add this line before the display call in the script:
Python
image = image.rotate(180, expand=True)
Then restart the service:
sudo systemctl restart rklb-ticker.service
Useful Commands
View logs: tail -f /var/log/rklb-ticker.log
Restart ticker: sudo systemctl restart rklb-ticker.service
Stop ticker: sudo systemctl stop rklb-ticker.service
Acknowledgments
Waveshare for the excellent e-Paper HAT
yfinance library for stock data
License
MIT License — feel free to use, modify, and share!











