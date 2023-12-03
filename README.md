# Free, simple, open recipe dataset
Simple recipe datasets with title of dish, ingredients, and instructions.

### 13k-recipes.csv
- CSV file with 13,000 recipes
- 26.6 mb

### 13k-recipes.db
- Database file with 13,000 recipes
- 26 mb

### 5k-recipes.db
- Database file with 5,000 recipes (the first 5,000 rows from 13k-recipes.db)
- 9.9 mb

## Source
The original dataset was created by scraping from the Epicurious Website. The content was uploaded to Kaggle as [Food Ingredients and Recipes Dataset with Images](https://www.kaggle.com/datasets/pes12017000148/food-ingredients-and-recipe-dataset-with-images). 


## Content
The orignal dataset was over 200 mb and contained 13,500 recipes and images. I dropped the images and created the simplified resources in this repo. No changes have been made to the text content.

Each database contains a "recipes" table with the following three columns:
- Title: Title of the dish.
- Ingredients: Ingredients as they were scraped from the website.
- Instructions: Instructions to recreate the dish.

### Database schema:
```
CREATE TABLE "recipes" (
    [id] INTEGER PRIMARY KEY,
    [Title] TEXT,
    [Ingredients] TEXT,
    [Instructions] TEXT
);
```

### Example row:
| id | Title | Ingredients | Instructions |
| --- | --- | --- | --- |
| 29 | Baigan Chokha | ['2 large Italian eggplants', '1 tablespoon canola oil', '½ medium onion, chopped', '2 cloves garlic, finely chopped', '1 small tomato, chopped', '¼ teaspoon coarse salt, or to taste', 'Freshly ground black pepper to taste', '1 tablespoon coarsely chopped cilantro', 'Roti, for serving'] | Prepare a hot grill or preheat the broiler.<br>With a fork, pierce the eggplants all over, and place on the grill or under the broiler. Grill or broil until completely charred and soft, about 20 minutes, turning frequently (the eggplants will brown and blister quickly). Remove and allow to cool. <br>Once cool, cut open the eggplants and scrape out the flesh. The flesh should be soft to the touch and pulpy, and should easily come away from the skin. Set aside. <br>Heat the canola oil in a frying pan. Add the onion and sauté until translucent. Add the garlic and fry until the garlic turns a dark golden brown, then add the tomato and fry for 1 to 2 minutes.<br>Stir in the mashed eggplant and cook for about 2 minutes. Season with salt and black pepper to taste.<br>Garnish with the cilantro and serve with roti. |

## License:
[CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/)