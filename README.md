import fastf1
import fastf1.plotting
import pandas as pd
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os

# Initialize FastF1 plotting for official colors
fastf1.plotting.setup_mpl(misc_mpl_mods=False)

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Setup Caching
cache_dir = 'fastf1_cache'
if not os.path.exists(cache_dir):
    os.makedirs(cache_dir)
fastf1.Cache.enable_cache(cache_dir)

@app.get("/telemetry/{year}/{gp}")
def get_f1_data(year: int, gp: str):
    try:
        # 1. Load the session
        # 'gp' will be 'bahrain', 'saudi', etc., from your HTML dropdown
        session = fastf1.get_session(year, gp, 'R')
        session.load(laps=True, telemetry=False, weather=False)
        
        # 2. Get quick laps to avoid pit stop spikes in your graph
        laps = session.laps.pick_quicklaps()
        
        response_data = {}
        
        # 3. Group by Driver
        for driver_code, data in laps.groupby('Driver'):
            # Get the official team color
            try:
                color = fastf1.plotting.driver_color(driver_code)
            except:
                color = "#ffffff" # Fallback to white if color fetch fails

            response_data[driver_code] = {
                "laps": data['LapNumber'].astype(int).tolist(),
                "times": data['LapTime'].dt.total_seconds().tolist(),
                "color": color
            }
            
        return response_data
    except Exception as e:
        print(f"Error: {e}")
        return {"error": str(e)}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)# backend-app.py