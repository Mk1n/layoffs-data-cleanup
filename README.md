# layoffs-data-cleanup
Project to clean and standardize layoffs data
-- Data Cleaning

SELECT * 
FROM layoffs;

-- 1. Remove Duplicates
-- 2. Standardize the data
-- 3. Look for null values or blanks 
-- 4. Remove any columns and rows that are not necessary - there are a few ways to do this

-- Create a new table that mirrors the original layout but is empty
CREATE TABLE layoffs_staging
LIKE layoffs;

-- Check the new staging table
SELECT * 
FROM layoffs_staging;

-- Copy data from the original table to the staging table
INSERT layoffs_staging
SELECT *
FROM layoffs;

-- Explanation
-- I created a new table that matches the original table structure but is empty. 
-- I then copied the data into it using the INSERT statement. 
-- The benefit is that I want to clean and modify the data without altering the original table directly.

-- Identify duplicates with row numbers
SELECT *, 
ROW_NUMBER() OVER(
PARTITION BY company, industry, total_laid_off, `date` ORDER BY `date` ASC) AS row_num
FROM layoffs_staging;

-- View the staging table
SELECT *
FROM world_layoffs.layoffs_staging;

-- Using a CTE to identify duplicates
WITH duplicate_cte AS (
    SELECT *,
    ROW_NUMBER() OVER (
        PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
    ) AS row_num
    FROM layoffs_staging
) 
SELECT *
FROM duplicate_cte
WHERE row_num > 1;

-- Another approach to view duplicates
SELECT *
FROM (
    SELECT company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions,
        ROW_NUMBER() OVER (
            PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
        ) AS row_num
    FROM world_layoffs.layoffs_staging
) duplicates
WHERE row_num > 1;

-- Now, you may want to write it like this to delete duplicates:
WITH DELETE_CTE AS (
    SELECT *
    FROM (
        SELECT company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions,
            ROW_NUMBER() OVER (
                PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
            ) AS row_num
        FROM layoffs_staging
    ) duplicates
    WHERE row_num > 1
)
DELETE
FROM DELETE_CTE;

-- Create a second staging table
CREATE TABLE `layoffs_staging2` (
    `company` text,
    `location` text,
    `industry` text,
    `total_laid_off` INT,
    `percentage_laid_off` text,
    `date` text,
    `stage` text,
    `country` text,
    `funds_raised_millions` int DEFAULT NULL,
    `row_num` int
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- View the second staging table for duplicates
SELECT *
FROM layoffs_staging2
WHERE row_num > 1;

-- Insert into the second staging table from the first
INSERT INTO layoffs_staging2 
SELECT company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions,
ROW_NUMBER() OVER (
    PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
) AS row_num
FROM layoffs_staging;

-- Remove duplicates from the second staging table
DELETE
FROM layoffs_staging2
WHERE row_num > 1;

-- View the cleaned second staging table
SELECT *
FROM layoffs_staging2;

-- 2. Standardize the data

-- View distinct company names
SELECT DISTINCT(TRIM(company))
FROM layoffs_staging2;

-- Remove extra spaces from company names
UPDATE layoffs_staging2
SET company = TRIM(company);

-- View distinct industry values
SELECT DISTINCT industry
FROM layoffs_staging2;

-- View rows where industry matches 'Crypto'
SELECT *
FROM layoffs_staging2
WHERE industry LIKE 'Crypto%';

-- Standardize industry names
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

-- View country names matching 'United States'
SELECT *
FROM layoffs_staging2
WHERE country LIKE 'United States%'
ORDER BY country;

-- Clean country names by removing trailing periods
SELECT DISTINCT country, TRIM(TRAILING '.' FROM country) 
FROM layoffs_staging2
ORDER BY country;

UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country) 
WHERE country LIKE 'United States%';

-- Convert date strings to actual date format
SELECT `date`, STR_TO_DATE(`date`, '%m/%d/%Y') 
FROM layoffs_staging2;

-- Update the date format in the staging table
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

-- Check the updated date format
SELECT `date` 
FROM layoffs_staging2;

-- Alter the date column type to DATE
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;

-- View the cleaned second staging table again
SELECT *
FROM layoffs_staging2;

-- 3. Look for null values or blanks 

-- Check the second staging table for nulls
SELECT * 
FROM world_layoffs.layoffs_staging2;

-- Look at distinct industries to identify nulls
SELECT DISTINCT industry
FROM world_layoffs.layoffs_staging2
ORDER BY industry;

-- Find records with null or empty industry
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL OR industry = ''
ORDER BY industry;

-- Check specific company records
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE company LIKE 'Bally%';

-- Check for Airbnb records
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE company LIKE 'airbnb%';

-- It seems like Airbnb has missing industry information.
-- We can write a query to update null industry values based on existing company records.
-- This approach is efficient for a large dataset.

-- Set blank industry values to null for easier handling
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';

-- Verify that blanks have been converted to nulls
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL OR industry = ''
ORDER BY industry;

-- Populate null industry values based on the same company
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;

-- Check the records again to confirm population of null values
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL OR industry = ''
ORDER BY industry;

-- Delete rows where total_laid_off and percentage_laid_off are both NULL
DELETE 
FROM layoffs_staging2
WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;

-- View the second staging table again
SELECT *
FROM layoffs_staging2;

-- Remove unnecessary column 'row_num'
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
