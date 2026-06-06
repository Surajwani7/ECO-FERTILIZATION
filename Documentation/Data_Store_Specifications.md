# Data Store (DS) Specifications - Eco-Fertilization System

## Overview
This document defines all data stores used in the Eco-Fertilization system, including their structure, format, purpose, and access patterns.

---

## DS-1: InputData.csv

### Purpose
Store the user's input data (crop name, state, city) for the current recommendation session.

### Location
```
Code/app/InputData.csv
```

### File Format
```
CSV (Comma-Separated Values)
```

### Data Schema
```
crop_name,state_name,city_name
```

### Structure & Fields

| Field | Type | Size | Required | Description |
|-------|------|------|----------|-------------|
| crop_name | String | 50 chars | Yes | Name of the crop (e.g., "Rice") |
| state_name | String | 50 chars | Yes | Indian state name (e.g., "Karnataka") |
| city_name | String | 50 chars | Yes | City name (e.g., "Bangalore") |

### Example Data
```
Rice,Karnataka,Bangalore
Wheat,Punjab,Ludhiana
Corn,Madhya Pradesh,Indore
```

### Access Pattern
- **Write:** When user submits form (POST request)
- **Read:** When generating report
- **Write Frequency:** Once per recommendation
- **Overwrite Policy:** Overwrite previous data on new submission

### File Size
- **Typical Size:** 50-100 bytes
- **Max Size:** 200 bytes

### Data Validation Rules
1. Crop name must exist in supported crops list
2. State must be valid Indian state
3. City must exist in selected state
4. No null values allowed
5. Whitespace trimmed on save

### Implementation Code
```python
# Code/app/app.py - Lines 27-29
with open("InputData.csv", "w") as fh:
    input_data = "%s,%s,%s" % (crop.strip(), state.strip(), city.strip())
    fh.write(input_data)
```

### Access Code
```python
# Reading
with open("InputData.csv", "r") as fh:
    data = fh.read()
    crop, state, city = data.split(",")
```

---

## DS-2: Nutrient_recommendation.csv

### Purpose
Store the training dataset containing historical crop-weather-nutrient relationships for ML model training.

### Location
```
Code/app/Nutrient_recommendation.csv
Code/backend/Nutrient_recommendation.csv
```

### File Format
```
CSV (Comma-Separated Values)
No header row (header added programmatically)
```

### Data Schema
```
Crop,Temperature,Humidity,Rainfall,Label_N,Label_P,Label_K
```

### Structure & Fields

| Field | Type | Range | Unit | Description |
|-------|------|-------|------|-------------|
| Crop | String | - | - | Crop name (categorical) |
| Temperature | Float | 15-40 | °C | Average temperature |
| Humidity | Float | 30-100 | % | Relative humidity |
| Rainfall | Float | 0-300 | mm | Rainfall amount |
| Label_N | Float | 10-80 | kg/ha | Nitrogen requirement |
| Label_P | Float | 5-40 | kg/ha | Phosphorus requirement |
| Label_K | Float | 10-60 | kg/ha | Potassium requirement |

### Example Data
```
Rice,28,75,5.2,45.3,20.1,35.7
Wheat,22,65,2.1,38.5,18.2,28.9
Corn,25,70,4.5,42.1,19.5,32.4
Rice,29,80,8.5,48.2,22.3,38.1
Wheat,20,60,1.5,35.2,17.0,26.5
```

### File Size
- **Typical Size:** 100-150 KB
- **Number of Records:** 500-2000+
- **Record Size:** 50-60 bytes per record

### Data Characteristics
- **Multiple entries per crop:** Different weather conditions
- **Crop variety:** Rice, Wheat, Corn, Sugarcane, Cotton, Potato, Tomato, etc.
- **Balanced distribution:** Representative of various climate zones
- **Historical data:** Based on agronomic research

### Column Definitions

**Crop:**
- Categorical variable
- Values: Rice, Wheat, Corn, Cotton, Sugarcane, Potato, Tomato, Soybean, Barley, Oats
- Case-sensitive in matching

**Temperature:**
- Continuous numerical
- Range: 15-40°C
- Represents growing season average

**Humidity:**
- Continuous numerical
- Range: 30-100%
- Relative humidity percentage

**Rainfall:**
- Continuous numerical
- Range: 0-300mm
- Seasonal rainfall during growing period

**Label_N (Nitrogen):**
- Continuous numerical
- Range: 10-80 kg/hectare
- Target output for prediction

**Label_P (Phosphorus):**
- Continuous numerical
- Range: 5-40 kg/hectare
- Target output for prediction

**Label_K (Potassium):**
- Continuous numerical
- Range: 10-60 kg/hectare
- Target output for prediction

### Access Pattern
- **Write:** Manual updates when training data changes
- **Read:** During ML model training (every prediction cycle)
- **Read Frequency:** Once per user request
- **Write Frequency:** Quarterly or when data updated

### Implementation Code
```python
# Code/app/NPKEstimatorModule.py - Lines 14-24
class NPKEstimator:
    def __init__(self, data='Nutrient_recommendation.csv'):
        self.df = pd.read_csv(data, header=None)
        
    def renameCol(self):
        self.df.columns = ['Crop', 'Temperature', 'Humidity', 
                          'Rainfall', 'Label_N', 'Label_P', 'Label_K']
        self.df.drop(self.df.index[:1], inplace=True)  # Drop header row
```

### Data Quality Notes
- Missing values: None (complete dataset)
- Outliers: Validated and reasonable
- Duplicates: May exist for different crop-weather combinations
- Consistency: All values within expected agricultural ranges

---

## DS-3: mapped_crops.csv

### Purpose
Store the mapping between crop names and their numeric IDs for ML model encoding.

### Location
```
Code/app/mapped_crops.csv
```

### File Format
```
CSV (Comma-Separated Values)
With header row
```

### Data Schema
```
Crops,Key
```

### Structure & Fields

| Field | Type | Description |
|-------|------|-------------|
| Crops | String | Crop name (categorical) |
| Key | Integer | Numeric identifier for encoding |

### Example Data
```
Crops,Key
Rice,1
Wheat,2
Corn,3
Cotton,4
Sugarcane,5
Potato,6
Tomato,7
Soybean,8
Barley,9
Oats,10
NA,nan
```

### Access Pattern
- **Write:** Generated when ML model trains
- **Read:** During crop name encoding for predictions
- **Write Frequency:** Once per prediction cycle
- **Read Frequency:** Once per prediction

### File Size
- **Typical Size:** 200-400 bytes
- **Number of Records:** 10-15 crops + NA

### Data Purpose
- **Ordinal Encoding:** Convert categorical crop names to integers
- **Reversal:** Map back to crop names after prediction
- **NA Handling:** Map unknown crops to NaN

### Implementation Code
```python
# Code/app/NPKEstimatorModule.py - Lines 27-42
def cropMapper(self):
    mapping = dict()
    
    with open("mapped_crops.csv", "w") as fh:
        fh.write("Crops,Key\n")
        for i, crop in enumerate(np.unique(self.df[['Crop']]), 1):
            mapping[crop] = i
            fh.write("%s,%d\n" % (crop, i))
        mapping['NA'] = np.nan
        fh.write("NA,nan")
    
    ordinal_cols_mapping = [{"col": "Crop", "mapping": mapping}]
    encoder = ce.OrdinalEncoder(cols='Crop', mapping=ordinal_cols_mapping, 
                               return_df=True)
    return mapping, encoder
```

### Data Quality
- **Unique mappings:** Each crop has unique ID
- **Sequential:** IDs numbered 1, 2, 3, ...
- **Consistency:** Same crop always maps to same ID
- **Completeness:** All crops in training data mapped

---

## DS-4: weather_data.json

### Purpose
Store the real-time 7-day weather forecast data retrieved from Weatherbit API.

### Location
```
Code/app/weather_data.json
Code/backend/weather_data.json
```

### File Format
```
JSON (JavaScript Object Notation)
Pretty-printed with indentation for readability
```

### Data Schema
```json
{
  "city_name": "string",
  "data": [
    {
      "datetime": "YYYY-MM-DD",
      "temp": number,
      "rh": number,
      "precip": number,
      "pop": number,
      "weather": {
        "code": number,
        "description": "string",
        "icon": "string"
      }
    }
    // ... 7 entries for 7 days
  ]
}
```

### Example Data
```json
{
  "city_name": "Bangalore",
  "data": [
    {
      "datetime": "2026-06-07",
      "temp": 28.5,
      "rh": 75,
      "precip": 5.2,
      "pop": 45,
      "weather": {
        "code": 500,
        "description": "Light rain",
        "icon": "c04d"
      }
    },
    {
      "datetime": "2026-06-08",
      "temp": 26.3,
      "rh": 80,
      "precip": 12.5,
      "pop": 85,
      "weather": {
        "code": 502,
        "description": "Heavy rain",
        "icon": "c10d"
      }
    },
    // ... 5 more days
  ]
}
```

### Structure & Fields

| Field | Type | Range | Unit | Description |
|-------|------|-------|------|-------------|
| city_name | String | - | - | Name of the city |
| datetime | Date | YYYY-MM-DD | - | Forecast date |
| temp | Float | -20 to 50 | °C | Temperature |
| rh | Float | 0-100 | % | Relative humidity |
| precip | Float | 0-500 | mm | Precipitation amount |
| pop | Float | 0-100 | % | Probability of precipitation |
| code | Integer | - | - | Weather condition code |
| description | String | - | - | Human-readable weather description |
| icon | String | - | - | Weather icon code |

### Weather Code Reference
| Code | Description |
|------|-------------|
| 200 | Thunderstorm with light rain |
| 201 | Thunderstorm with rain |
| 202 | **Thunderstorm with heavy rain** |
| 233 | **Thunderstorm with heavy drizzle** |
| 300 | Light drizzle |
| 500 | Light rain |
| 501 | Moderate rain |
| 502 | **Heavy rain** |
| 520 | Light shower rain |
| 521 | **Ragged shower rain** |
| 522 | **Ragged heavy shower rain** |

**Heavy Rain Codes:** 202, 233, 502, 521, 522

### File Size
- **Typical Size:** 2-5 KB
- **Structure:** 1 object with 7 forecast entries
- **Record Size:** 250-400 bytes per day

### Access Pattern
- **Write:** After API call from Weatherbit
- **Read:** During weather analysis and risk assessment
- **Write Frequency:** Once per recommendation request
- **Read Frequency:** Multiple times (analysis, display)

### Data Source
- **External API:** Weatherbit API v2.0
- **Endpoint:** `https://api.weatherbit.io/v2.0/forecast/daily?`
- **Update Frequency:** Real-time (6-8 hours for forecast)
- **Accuracy:** Varies by location and time

### Implementation Code
```python
# Code/app/BestTimeToFertilizeModule.py - Lines 42-50
def json_file_bulider(self):
    try:
        json_obj = self.response.json()
        with open('weather_data.json', 'w') as file:
            js.dump(json_obj, file, indent=1, sort_keys=True)
        print("weather_data.json file build successfully")
    except Exception as msg:
        print("json_bulider():", msg)

# Access
json_obj = self.response.json()
for i in range(self.days):  # 7 days
    date = json_obj['data'][i]['datetime']
    temp = json_obj['data'][i]['temp']
    rh = json_obj['data'][i]['rh']
    precip = json_obj['data'][i]['precip']
```

### Data Quality
- **Completeness:** All 7 days guaranteed
- **Consistency:** All fields always present
- **Validity:** Codes from official Weatherbit list
- **Freshness:** Updated on each API call

---

## DS-5: output.txt

### Purpose
Store the final recommendation report for the current session.

### Location
```
Code/app/output.txt
```

### File Format
```
Plain text
Multiple lines, newline-separated
```

### Data Schema
```
[Category]
[Heading]
[Description]
[N_Value]
[P_Value]
[K_Value]
```

### Example Data
```
Warning
Heavy Rain Alert
Heavy Rain Chances within 2 days from now is 65%
45.3
20.1
35.7
```

### Structure & Fields

| Line | Field | Type | Description |
|------|-------|------|-------------|
| 1 | Category | String | Alert category: Warning, Message, Success |
| 2 | Heading | String | Alert type heading |
| 3 | Description | String | Detailed message to user |
| 4 | Label_N | Float | Nitrogen requirement (kg/ha) |
| 5 | Label_P | Float | Phosphorus requirement (kg/ha) |
| 6 | Label_K | Float | Potassium requirement (kg/ha) |

### Data Values

**Category Options:**
- `Warning` - Risk detected (heavy/prolonged rain)
- `Message` - General information (safe conditions)
- `Success` - Successful recommendation

**Heading Examples:**
- "Heavy Rain Alert"
- "Prolonged Rainfall Alert"
- "Precipitation Amount"
- "Safe to Fertilize"

**Description Examples:**
- "Heavy Rain Chances within 2 days from now is 65%"
- "Prolonged Rainfall of greater than 12.7 mm puts your fertilizer at risk"
- "The amount of rain for 2 days, counting today is 5.20 mm and chances is 45%"

**NPK Values:**
- Format: Floating point number with 1 decimal place
- Range: N (10-80), P (5-40), K (10-60)

### File Size
- **Typical Size:** 200-400 bytes
- **Fixed Structure:** Always 6 lines
- **Line Length:** 30-200 characters per line

### Access Pattern
- **Write:** After report generation (P5)
- **Read:** For display (P6) and record-keeping
- **Write Frequency:** Once per recommendation
- **Overwrite Policy:** Overwrite previous output

### Implementation Code
```python
# Code/app/app.py - Lines 58-61
output_data = category + "\n" + heading + "\n" + desc + "\n" + \
              str(npk['Label_N']) + "\n" + str(npk['Label_P']) + "\n" + \
              str(npk['Label_K'])
with open("output.txt", "w") as fh:
    fh.write(output_data)
```

### Data Quality
- **Completeness:** All 6 fields always present
- **Validation:** Values checked before write
- **Format:** Consistent newline separation
- **Encoding:** UTF-8 text format

---

## DS-6: System Data Structure (In-Memory)

### Purpose
Store processed data in Python dictionaries during runtime for web template rendering.

### Structure

```python
recommendation_data = {
    'call_success': [1],  # List with status flag
    'npk': [
        {
            'Label_N': 45.3,
            'Label_P': 20.1,
            'Label_K': 35.7
        }
    ],
    'form_data': {
        'crop': 'Rice',
        'state': 'Karnataka',
        'city': 'Bangalore'
    },
    'popup_data': [
        [
            'Warning',  # Category
            'Heavy Rain Alert',  # Heading
            'Heavy Rain Chances within 2 days from now is 65%'  # Description
        ]
    ],
    'seven_days': [
        {
            'Date': '2026-06-07',
            'Temperature': 28.5,
            'Relative Humidity': 75,
            'Rainfall': 5.2,
            'Probability of Precipitation': 45,
            'Weather Description': 'Light rain'
        },
        # ... 6 more days
    ]
}
```

### Access Pattern
- **Create:** In `app.py` processing() function
- **Pass:** To Flask's render_template()
- **Read:** By HTML template (Jinja2)
- **Lifetime:** Duration of HTTP response

### Implementation Code
```python
# Code/app/app.py - Lines 17-64
call_success = []
npk_list_dict = []
popup_data = []
seven_days = []

# ... process data ...

return render_template('update.html', 
                      CALL_SUCCESS=call_success,
                      NPK=npk_list_dict,
                      FORM_DATA=form_data,
                      POPUP_DATA=popup_data,
                      SEVEN_DAYS=seven_days)
```

---

## Data Store Access Patterns Summary

| Data Store | Read | Write | Update | Delete | Access Frequency |
|-----------|------|-------|--------|--------|------------------|
| DS-1 InputData.csv | P1,P5 | P1 | P1 | Manual | Once per request |
| DS-2 Nutrient_recommendation.csv | P4 | Manual | Manual | Manual | Once per request |
| DS-3 mapped_crops.csv | P4 | P4 | P4 | Manual | Once per request |
| DS-4 weather_data.json | P2,P3,P6 | P2 | P2 | Manual | Multiple per request |
| DS-5 output.txt | P6 | P5 | P5 | Manual | Once per request |
| DS-6 In-Memory | P6 | P5 | P5 | Auto | Once per request |

---

## Data Flow Diagram (DS Perspective)

```
User Input
    ↓
[DS-1: InputData.csv] ← P1 writes
    ↓
[DS-2: Nutrient_recommendation.csv] + [DS-3: mapped_crops.csv]
    ↓
P4 reads for ML training
    ↓
Weatherbit API → [DS-4: weather_data.json] ← P2 writes
    ↓
P3 reads for analysis
    ↓
[DS-5: output.txt] ← P5 writes
    ↓
[DS-6: In-Memory] ← Combined for display
    ↓
P6 renders HTML → User
```

---

## Data Validation Rules

### DS-1 (InputData.csv)
- Crop must be in supported list
- State must be valid Indian state
- City must exist in state
- No empty fields allowed
- Whitespace trimmed

### DS-2 (Nutrient_recommendation.csv)
- Numerical values in valid ranges
- No missing values
- Unique crop-weather-nutrient records
- All 7 columns present

### DS-3 (mapped_crops.csv)
- Sequential integer keys (1, 2, 3, ...)
- Unique crop names
- NA mapping for unknowns
- Header row present

### DS-4 (weather_data.json)
- Valid JSON format
- All 7 days present
- All required fields in each day
- Weather codes from official list
- Temperature, humidity, rainfall ranges valid

### DS-5 (output.txt)
- 6 lines exactly
- Numeric values formatted correctly
- Category is one of: Warning, Message, Success
- NPK values positive

### DS-6 (In-Memory)
- All required keys present
- Correct data types
- Non-empty lists/dicts
- Ready for template rendering

---

## Data Security Considerations

### File-Based Data Stores (DS-1 to DS-5)
- **Access Control:** File system permissions
- **Encryption:** Not currently implemented
- **Backup:** Manual backups recommended
- **Sensitivity:** Low (agricultural data)

### In-Memory Data (DS-6)
- **Scope:** Single HTTP request
- **Cleanup:** Automatic after response
- **Exposure:** None (not persisted)
- **Sensitivity:** Low

### Weatherbit API Data (DS-4)
- **Source:** External trusted service
- **API Key:** Hardcoded (consider environment variables)
- **HTTPS:** Yes, encrypted in transit
- **Storage:** Not sensitive, can be cached

---

## Performance Considerations

| Data Store | Read Time | Write Time | Size | Notes |
|-----------|-----------|-----------|------|-------|
| DS-1 | <10ms | <10ms | <1KB | Fast text I/O |
| DS-2 | 100-200ms | Manual | 100-150KB | Large CSV, pandas I/O |
| DS-3 | <10ms | <10ms | <1KB | Small, fast lookup |
| DS-4 | 10-50ms | 50-100ms | 2-5KB | API JSON, parsing overhead |
| DS-5 | <10ms | <10ms | <1KB | Fast text I/O |
| DS-6 | <1ms | 10-50ms | <10KB | In-memory, fastest |

### Optimization Opportunities
1. **DS-2:** Cache loaded CSV in memory (singleton pattern)
2. **DS-4:** Implement weather data caching
3. **DS-3:** Pre-load mapping at startup
4. **DS-5:** Consider database instead of file

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-06 | Initial comprehensive data store specification |

---

**This document provides complete data store specifications for the Eco-Fertilization system.**
