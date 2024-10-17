# JOBS ANALYSIS

![Jobs Analysis](https://github.com/user-attachments/assets/ea96b3d3-ca29-4d51-92ff-85a35e922f35)

This repository presents a comprehensive analysis of job data extracted from the Naukari website. The project aims to gather over 700 job details using the Python library Selenium, systematically organizing the information for further analysis. It includes data cleaning and exploratory data analysis (EDA) using Pandas and NumPy, while MySQL and MS Excel were utilized for insights, and Power BI was employed for visualization.

## Workflow

1. **Data Scraping**: Use the [Naukari Web Scraper](https://github.com/sameermujahid/Naukari-web-scraper) to extract job data from the Naukari website.
2. **Data Validation**: Cleaned and validated the scraped data to ensure accuracy and consistency.
3. **Exploratory Data Analysis (EDA)**: Conducted univariate, bivariate, and multivariate analyses to uncover patterns in the dataset.
4. **Data Visualization**: Developed visual representations of the data using Power BI.

## Tools & Technology Used
![Tools and Technologies](https://user-images.githubusercontent.com/55955478/235897588-756e9ed4-33b3-45f6-83c8-4f68988fe8ba.png)

### Data Collection
Utilized web scraping with the Python library Selenium to extract and retrieve the following attributes:

| Column             | Meaning                                                                                          |
|--------------------|--------------------------------------------------------------------------------------------------|
| Job ID             | Unique identifier for each job listing.                                                         |
| Job Title          | Title of the job position.                                                                       |
| Company            | Name of the company offering the job.                                                            |
| Reviews            | Average reviews or ratings for the company (if available).                                      |
| Location           | Geographical location of the job.                                                               |
| Experience         | Required experience level for the job (e.g., years of experience).                             |
| Salary             | Salary range or figure associated with the job position.                                        |
| Posted On          | Date when the job was posted.                                                                    |
| Openings           | Number of job openings available for the position (if available).                               |
| Applications       | Number of applications received for the job listing.                                            |
| Job Description    | Detailed description of job responsibilities and requirements.                                   |
| Role               | Specific role associated with the job (if specified).                                           |
| Industry Type      | Type of industry the company operates in (e.g., IT, Healthcare).                               |
| Department         | Department within the company relevant to the job position (e.g., Marketing, Engineering).     |
| Employment Type    | Type of employment (e.g., Full-time, Part-time, Contract).                                     |
| Role Category      | Classification of the job role (e.g., Manager, Intern).                                        |
| Education          | Educational qualifications required for the job.                                               |
| Key Skills         | Essential skills required to perform the job effectively.                                       |

## Data Preprocessing

The script processes job listing data to clean and structure it for analysis. The main tasks include cleaning company text, normalizing reviews, extracting experience and salary information, and standardizing educational qualifications. Below is a detailed explanation of each preprocessing step.

### Import Required Libraries
```python
import pandas as pd
import numpy as np
import re
from datetime import datetime, timedelta
```

### Clean Company Text
```python
def clean_company_text(company_text):
    cleaned_text = re.sub(r"\n\nemployees' choice", '', company_text)
    return cleaned_text.strip()

df['Company'] = df['Company'].apply(clean_company_text)
```

### Round Reviews and Handle Missing Values
```python
df['Reviews'] = df['Reviews'].round(1)
mode_value = df['Reviews'].mode()[0]  # Uncomment to fill missing reviews
# df['Reviews'].fillna(mode_value, inplace=True)
```

### Extract Experience Data
```python
df['Min_Experience'] = df['Experience'].str.split(' - ').str[0].str.replace(' years', '', regex=False).astype(float).fillna(0).astype(int)
df = df[df['Experience'].str.contains(r'\d{1,2} - \d{1,2} years| \d{1,2} years', na=False)]
df['Max_Experience'] = df['Experience'].str.split(' - ').str[1].str.replace(' years', '', regex=False).fillna('0').astype(int)
```

### Convert and Clean Salary Data
```python
def convert_salary(salary):
    salary_replacements = {
        '50,000': '0.5',
        '60,000': '0.6',
        '70,000': '0.7'
    }
    for old_salary, new_salary in salary_replacements.items():
        if re.search(rf'\b{old_salary}\b', salary):
            return salary.replace(old_salary, new_salary)
    return salary

df['Salary'] = df['Salary'].apply(convert_salary)

def clean_salary(salary):
    if isinstance(salary, str):
        if 'not disclosed' in salary.lower():
            return np.nan, np.nan, np.nan
        if 'p.a.' in salary:
            salary = salary.replace('p.a.', '').strip().replace(',', '')
        if 'lacs' in salary:
            salary = salary.replace('lacs', '').strip()
        if '-' in salary:
            min_salary, max_salary = salary.split('-')
            min_salary = float(min_salary) * 100000
            max_salary = float(max_salary) * 100000
            average_salary = (min_salary + max_salary) / 2
            return min_salary, max_salary, average_salary
        try:
            salary_value = float(salary) * 100000
            return salary_value, salary_value, salary_value
        except ValueError:
            return np.nan, np.nan, np.nan
    return np.nan, np.nan, np.nan

df[['Min_Salary', 'Max_Salary', 'Average_Salary']] = df['Salary'].apply(clean_salary).apply(pd.Series)
```

### Extract Posted Date Information
```python
def extract_days(posted_on):
    posted_on = posted_on.replace(' days ago', '').replace(' day ago', '').strip()
    if '30+' in posted_on:
        return 31
    if posted_on.isdigit():
        return int(posted_on)
    return np.nan

def calculate_date_from_days(days):
    if pd.isna(days):
        return np.nan
    reference_date = datetime.strptime('10-09-2024', '%d-%m-%Y')
    return (reference_date - timedelta(days=days)).strftime('%d-%m-%Y')

df['Days Posted On'] = df['Posted On'].apply(extract_days).fillna(0).astype(int)
df['Date Posted'] = df['Days Posted On'].apply(calculate_date_from_days)
```

### Clean Applications Data
```python
def clean_applications(value):
    if isinstance(value, str) and 'less than' in value:
        return int(re.findall(r'\d+', value)[0]) - 1
    try:
        return int(value)
    except ValueError:
        return np.nan

df['Applications'] = df['Applications'].apply(clean_applications)
df['Applications'] = df['Applications'].astype('Int64')
```

### Education Data
```python
def parse_education(row):
    ug, pg, doctorate = None, None, None
    if isinstance(row, str):
        for entry in row.split('\n'):
            entry = entry.lower().strip()
            if entry.startswith('ug:'):
                ug = entry.replace('ug:', '').strip()
            elif entry.startswith('pg:'):
                pg = entry.replace('pg:', '').strip()
            elif entry.startswith('doctorate:'):
                doctorate = entry.replace('doctorate:', '').strip()
    return pd.Series([ug, pg, doctorate])

df[['UG', 'PG', 'Doctorate']] = df['Education'].apply(parse_education)

def standardize_ug(qualification):
    if pd.isna(qualification):
        return 'Not Specified'
    qualification = qualification.lower()
    if 'b.tech' in qualification or 'b.e.' in qualification:
        return 'B.Tech/B.E.'
    # Add other qualifications as needed
    return 'Not Specified'

def standardize_pg(qualification):
    if pd.isna(qualification):
        return 'Not Specified'
    qualification = qualification.lower()
    if 'm.tech' in qualification:
        return 'M.Tech'
    # Add other qualifications as needed
    return 'Not Specified'

def standardize_doctorate(qualification):
    if pd.isna(qualification):
        return 'Not Specified'
    qualification = qualification.lower()
    if 'ph.d' in qualification or 'doctorate' in qualification:
        return 'Ph.D/Doctorate'
    return 'Not Specified'

df['UG'] = df['UG'].apply(standardize_ug)
df['PG'] = df['PG'].apply(standardize_pg)
df['Doctorate'] = df['Doctorate'].apply(standardize_doctorate)
```

### Clean Location Data
```python
df['Location'] = df['Location'].str.split(', ')
df = df.explode('Location')
df['Location'] = df['Location'].str.strip().str.lower()

def clean_location(location):
    location =

 re.sub(r'\(.*?\)', '', location).strip()
    return location

df['Location'] = df['Location'].apply(clean_location)

# Aggregate unique locations into a list per Job ID
aggregated_locations = df.groupby('Job ID')['Location'].apply(lambda x: list(sorted(set(x)))).reset_index()
aggregated_locations['Location'] = aggregated_locations['Location'].apply(tuple)
df = df.drop(columns=['Location']).merge(aggregated_locations, on='Job ID', how='left')
```

### Clean Key Skills Data
```python
df['Key Skills'] = df['Key Skills'].str.split(', ')
df = df.explode('Key Skills')

def clean_skill(skill):
    return skill.strip().lower()

df['Key Skills'] = df['Key Skills'].apply(clean_skill)

aggregated_skills = df.groupby('Job ID')['Key Skills'].apply(lambda x: list(sorted(set(x)))).reset_index()
df = df.drop(columns=['Key Skills']).merge(aggregated_skills, on='Job ID', how='left')
```

## Analysis

### Top Categories Analysis
![Top Categories in Roles](https://github.com/user-attachments/assets/6d1c7e9b-5754-479e-aea0-497fcd01f3c6)

![Top Categories in Companies](https://github.com/user-attachments/assets/33fa5533-9b80-4c6d-91da-6fbb81cd9ca8)

![Top Categories in Locations](https://github.com/user-attachments/assets/6c2921b9-cca6-46e0-b788-01a06643b3af)

![Top Categories in Skills](https://github.com/user-attachments/assets/277ae6c6-61dc-40c1-9542-2eb4b2814539)

![Top Categories in Role Categories](https://github.com/user-attachments/assets/587564d3-62f1-4652-9eb4-136c8715c853)

### Salary and Employment Analysis
![Max Salary by Industry Type](https://github.com/user-attachments/assets/65c02754-5fff-4533-b4f4-aa8c5d37bd03)

![Average Openings for Employment Type](https://github.com/user-attachments/assets/47a516fe-6182-41f5-9d15-a7d003890111)

### Skills Requirement Analysis
![Skills Needed for Top 10 Roles](https://github.com/user-attachments/assets/463bbc3e-2af5-460a-8d42-6636f36b329e)

## Power BI

An interactive Power BI dashboard has been developed to consolidate data from multiple sources. It showcases visually engaging charts, graphs, and tables, enabling users to explore key metrics and extract valuable insights. The dashboard enhances data-driven decision-making and facilitates effective communication with stakeholders.

![image](https://github.com/user-attachments/assets/df6d0e2b-b4e6-4d33-8cc7-9432c53cce8b)


## Job Market Analysis Conclusion

### Key Insights
#### Uni-Variate Analysis:
- **Job Roles**: Full Stack Developer and DevOps Engineer (35.51%) are in high demand, indicating job diversity with 50 unique roles.
- **Companies**: Accenture leads in postings, highlighting major tech companies as top employers among 461 total companies.
- **Locations**: Bengaluru and Hyderabad dominate (40.03%), reflecting the concentration of IT industries.
- **Salaries**: A significant portion (79.53%) of salaries are "Not Disclosed," complicating analysis, though high-paying outliers exist.
- **Employment Type**: Full-time permanent roles (92.90%) are predominant, with minimal flexible or freelance roles.
- **Skills & Education**: Python is the top skill, indicating high technical demand; "UG: Any Graduate" is common, but specific degrees are favored for technical roles.

#### Bi-Variate Analysis:
- **Salary and Industry Type**: IT Services and Consulting offer the highest salaries; part-time roles have competitive pay, but permanent roles are generally higher.
- **Experience and Salary**: There is a weak correlation between experience and salary, suggesting skills or industry type may be more influential.
- **Role and Key Skills**: Specialized roles require specific skills; for instance, Python and Machine Learning for data roles, and cloud technologies for DevOps.
- **Location and Role**: Larger cities offer more tech roles, while smaller cities present limited opportunities, indicating regional job disparities.
- **Educational Background**: Engineering degrees (B.Tech/B.E.) are preferred for technical roles, while generalist roles accept "Any Graduate."
- **Experience and Role**: DevOps Engineer roles often require higher experience (up to 14 years), while Full Stack Developer roles frequently hire less experienced candidates.

---
