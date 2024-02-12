COVID-19 Data Exploration Project
Overview.
This project involves the exploration of COVID-19 data using SQL queries. The queries leverage various SQL features such as Joins, Common Table Expressions (CTE), Temporary Tables, Windows Functions, and Aggregate Functions. The project aims to provide insights into the impact of COVID-19 on different countries and continents, including death rates, infection rates, vaccination statistics, and more.


/*
Covid 19 Data Exploration 

Skills used: Joins, CTE's, Temp Tables, Windows Functions, Aggregate Functions, Creating Views, Converting Data Types

 */
Select
  *
From
  CovidDeaths
Where
  continent is not null
order by
  3,
  4;

-- Select Data that we are going to be starting with
Select
  Location,
  date_2,
  total_cases,
  new_cases,
  total_deaths,
  population
From
  CovidDeaths
Where
  continent is not null
order by
  1,
  2;

-- Q1. Total Deaths / Total Cases
-- Show the chances of dying from Covid-19 in each country 
select
  location,
  date_2,
  total_cases,
  total_deaths,
  (total_deaths / total_cases) * 100 as Death_Percentage
from
  coviddeaths
where
  continent != ''
order by
  1,
  2;

-- Q2. Total Cases vs Population
-- Shows what percentage of population got infected with Covid-19
select
  location,
  date_2,
  total_cases,
  population,
  (total_cases / population) * 100 as Infection_rate
from
  coviddeaths
where
  continent != ''
order by
  1,
  2;

-- Q3. Countries with Highest Infection Rate compared to Population
-- create table Highest_Infection_rate as
select
  location,
  population,
  max(total_cases) Highest_Infection_Count,
  Max((total_cases / population) * 100) as Highest_Infection_rate
from
  coviddeaths
where
  continent != ''
group by
  location,
  population
order by
  Highest_Infection_rate desc;

-- Q4. Countries with Highest Death Count per Population
select
  location,
  max(cast(total_deaths as unsigned)) as Total_death_count
from
  coviddeaths
where
  continent != ''
group by
  location
order by
  Total_death_count desc;

-- Q5. BREAKING THINGS DOWN BY CONTINENT
-- Showing contintents with the highest death count per population
-- create table total_deaths_continent as
select
  continent,
   SUM(CAST(COALESCE(NULLIF(total_deaths, ''), '0') AS UNSIGNED)) AS Total_death_count
from
  coviddeaths
where
  continent != ''
group by
  continent
order by
  Total_death_count desc;

-- Q6. GLOBAL NUMBERS
-- create table Total_cases_and_deaths as  
Select
  SUM(new_cases) as total_cases,
  SUM(new_deaths) as total_deaths,
  SUM(new_deaths) / SUM(New_Cases) * 100 as DeathPercentage
From
  CovidDeaths
where
  continent is not null
order by
  1,
  2;

-- Q7. Vaccinations vs Total Population
-- Shows Percentage of Population that has recieved at least one Covid Vaccine.
Select
  coviddeaths.continent,
  coviddeaths.location,
  coviddeaths.date,
  coviddeaths.population,
  covidvaccinations.new_vaccinations,
  SUM(
    Cast(covidvaccinations.new_vaccinations as unsigned)
  ) OVER (
    Partition by
      coviddeaths.Location
    Order by
      coviddeaths.location,
      coviddeaths.Date
  ) as RollingPeopleVaccinated
From
  CovidDeaths
  Join CovidVaccinations On coviddeaths.location = covidvaccinations.location
  and coviddeaths.date = covidvaccinations.date
where
  coviddeaths.continent is not null
order by
  2,
  3;

-- Q8. Using CTE to perform Calculation on Partition By location in previous query  
with
  Pop_vs_Vac (
    continent,
    location,
    date_2,
    population,
    New_vaccination,
    RollingPeopleVaccinated
  ) as (
    select
      coviddeaths.continent,
      coviddeaths.location,
      coviddeaths.date_2,
      coviddeaths.population,
      covidvaccinations.new_vaccinations,
      sum(
        cast(covidvaccinations.new_vaccinations as unsigned)
      ) OVER (
        Partition by
          coviddeaths.location
        Order by
          coviddeaths.location,
          coviddeaths.date_2
      ) as RollingPeopleVaccinated
    from
      coviddeaths
      join covidvaccinations on coviddeaths.location = covidvaccinations.location
      and coviddeaths.date_2 = covidvaccinations.date_2
    where
      coviddeaths.continent != ''
  )
select
  *,
  (RollingPeopleVaccinated / Population) * 100
From
  Pop_vs_Vac;

-- Using Temp Table to perform Calculation on Partition By in previous query
DROP TABLE IF EXISTS
  PercentPopulationVaccinated;

CREATE TABLE
  PercentPopulationVaccinated (
    Continent NVARCHAR (255),
    Location NVARCHAR (255),
    Date_2 DATETIME,
    Population NUMERIC,
    New_vaccinations NUMERIC,
    RollingPeopleVaccinated NUMERIC
  );

INSERT INTO
  PercentPopulationVaccinated
SELECT
  coviddeaths.continent,
  coviddeaths.location,
  coviddeaths.date_2,
  coviddeaths.population,
  CASE
    WHEN TRIM(CovidVaccinations.new_vaccinations) = '' THEN 0
    ELSE CAST(CovidVaccinations.new_vaccinations AS SIGNED)
  END AS New_vaccinations,
  SUM(
    CAST(
      COALESCE(
        NULLIF(TRIM(CovidVaccinations.new_vaccinations), ''),
        '0'
      ) AS UNSIGNED
    )
  ) OVER (
    PARTITION BY
      coviddeaths.Location
    ORDER BY
      coviddeaths.location,
      coviddeaths.Date_2
  ) AS RollingPeopleVaccinated
FROM
  CovidDeaths
  JOIN CovidVaccinations ON coviddeaths.location = covidvaccinations.location
  AND coviddeaths.date_2 = covidvaccinations.date_2;

SELECT
  *,
  (RollingPeopleVaccinated / Population) * 100 AS VaccinationPercentage
FROM
  PercentPopulationVaccinated;

-- Creating View to store data for later visualizations
-- CREATE table PercentPopulationVaccinated AS
SELECT
  coviddeaths.continent,
  coviddeaths.location,
  coviddeaths.date_2,
  coviddeaths.population,
  CASE
    WHEN TRIM(covidvaccinations.new_vaccinations) = '' THEN 0
    ELSE CAST(covidvaccinations.new_vaccinations AS SIGNED)
  END AS new_vaccinations,
  SUM(
    CAST(
      COALESCE(
        NULLIF(TRIM(covidvaccinations.new_vaccinations), ''),
        '0'
      ) AS UNSIGNED
    )
  ) OVER (
    PARTITION BY
      coviddeaths.location
    ORDER BY
      coviddeaths.location,
      coviddeaths.date_2
  ) AS RollingPeopleVaccinated
FROM
  CovidDeaths
  JOIN CovidVaccinations ON coviddeaths.location = covidvaccinations.location
  AND coviddeaths.date = covidvaccinations.date
WHERE
  coviddeaths.continent IS NOT NULL;
  
  -- Percentage_population_infected by location and date
  CREATE TABLE Percentage_population_infected AS SELECT location,
    population,
    date_2,
    MAX(total_cases) AS Highest_infection_count,
    MAX((total_cases / population)) * 100 AS Percentage_population_infected FROM
    coviddeaths
GROUP BY location , population , date_2
ORDER BY Percentage_population_infected DESC;
  
  
  
  
