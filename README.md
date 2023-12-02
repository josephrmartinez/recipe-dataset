# Recipe Dataset
Simple recipe tables with title of dish, ingredients, and instructions.

*13k-recipes.csv*
- CSV file with 13,000 recipes
- 26.6 mb

13k-recipes.db
- Database file with 13,000 recipes
- 26 mb

5k-recipes.db
- Database file with 5,000 recipes
- This database contains the first 5,000 rows from 13k-recipes.db
- 9.9 mb

Source
The original dataset was uploaded to Kaggle as [Food Ingredients and Recipes Dataset with Images](https://www.kaggle.com/datasets/pes12017000148/food-ingredients-and-recipe-dataset-with-images). This original dataset was created by scraping from the Epicurious Website.

License: 
[CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/)

Content
The orignal dataset was over 200 mb and contained 13,500 recipes and images. I dropped the images and created the following resources:



Each database contains a "recipes" table with the following three columns:
Title: Title of the dish.
Ingredients: Ingredients as they were scraped from the website.
Instructions: Instructions to recreate the dish.

Database schema:

CREATE TABLE "recipes" (
    [id] INTEGER PRIMARY KEY,
    [Title] TEXT,
    [Ingredients] TEXT,
    [Instructions] TEXT
);
