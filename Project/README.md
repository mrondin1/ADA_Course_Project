# Analysing taste similarities in different cuisines

# mike note

Add data from users table in prod database for embedding. Export result as CSV, then upload into 'data/users/'.

Manually change curly brackets to square, handle in code later.

Data should look like:

```
user_id,user_name,ingredients
388,Apple John,"{""Baby Radish"",""Black Garlic"",Cauliflower,""Cauliflower Steak"",Chicken,""Dover Sole"",""Fingerling Potato"",""Fuji Apple Relish"",""Garlic Chive"",Herbs,Lamb,Oregano,""Parmesan Cheese"",""Parsnip Mousseline"",Rigatoni,""Rigatoni With Braised Lamb"",""Roasted Chicken"",""Sautéed Dover Sole With Fuji Apple Relish"",Shallot,""Toasted Almonds""}"
```

Use this SQL to get the data to export.

```
-- For embedding the users into our ingredient-based embedding space for new meal recs
-- Rough, can improve data cleanliness in future

-- Dedupped, single array, and grabbing from all possible swipes (ingredients or meals --> ingredients)
SELECT
	user_id,
	user_name,
	( -- deduplicate wrapper
		SELECT
			array_agg(DISTINCT val) AS ingredients
		FROM (
			SELECT
				unnest(ingredients) AS val) AS u)
	FROM (
		-- concatenate various ingredients wrappers
		SELECT
			user_id,
			user_name,
			array_cat(ingredients_array, array_cat(meals_array, meal_ingredients_array)) AS ingredients
		FROM (
			-- join all posible positive swipe ingredients (direct or thru meal, agg to user, drop nulls)
			SELECT
				users.id AS user_id,
				users.first_name || ' ' || users.last_name AS user_name,
				count(swiper_images.meal_id) AS meal_like_swipe_count,
				count(swiper_images.ingredient_id) AS ingredient_like_swipe_count,
				array_remove(array_agg(i1.name), NULL) AS ingredients_array,
				array_remove(array_agg(meals.name), NULL) AS meals_array,
				array_remove(array_agg(i2.name), NULL) AS meal_ingredients_array
			FROM
				client_preference_image_swipes
				JOIN swiper_images ON swiper_images.id = client_preference_image_swipes.swiper_image_id
				JOIN client_preferences cp ON cp.id = client_preference_image_swipes.client_preferences_id
				JOIN users ON users.id = cp.user_id
				LEFT JOIN ingredients i1 ON i1.id = swiper_images.ingredient_id
				LEFT JOIN meals ON meals.id = swiper_images.meal_id
				LEFT JOIN ingredient_lists ON ingredient_lists.meal_id = meals.id
				LEFT JOIN ingredients i2 ON i2.id = ingredient_lists.ingredient_id
			WHERE
				client_preference_image_swipes.like = TRUE
			GROUP BY
				users.id,
				user_name) raw
		GROUP BY
			user_id,
			user_name,
			ingredients_array,
			meals_array,
			meal_ingredients_array) x;
```

## Blog article

https://alioben.github.io/yummly/

# Abstract

Diets and food habits may vary widely from country to country in terms of ingredients and cooking techniques. Nevertheless, dishes from different regions may share similarities in flavours and tastes. This project aims to help people discover new recipes matching their preferences. Our analysis will be based on Yummly, a platform mainly used in North America containing recipes from different countries. First, we will characterise the taste preferences in a group of countries along with the nutritional intake of an average dish. Second, we will analyse similarities between cuisines in terms of flavours with the purpose of building a recipe recommender that matches one’s taste.

# How to run the Notebook

1. Clone the repository using `git clone <name repo>`
2. Download the dataset from [here](https://drive.google.com/open?id=18IHx-7FdWY9TdR4yHG2g-t1i0qAzdXOy) and unzip it under the folder data
3. Launch `jupyter notebook` and navigate to the cloned folder
4. Open `Final notebook.ipynb` to see the analysis

# Research questions

A list of research questions we would like to address during the project:

- What are the most commomly used ingredients?
- What are the most distinctive ingredients?
- How can we represent cuisines as a network of ingredients ?
- How can we cluster cuisine in terms of their recipe components?
- How do cuisines influence each other?

# Dataset

Yummly provides an API (https://developer.yummly.com/intro) with a limited number of calls for academic projects. We already sent a request to get access to it. The limit is set to 30K calls in total, therefore we may need to build a web crawler to scrap the data.
Yummly’s platform provides a search engine which filters the recipes by cuisine origin (e.g. Japanese, Italian…) and flavours (e.g. sweet, salty…). For each recipe, Yummly provides its ingredients, nutritional facts and some tags characterising the dish.
The Yummly API provides the following endpoints for searching and fetching recipes:

- `GET` requests can be done in the following way: `http://api.yummly.com/v1/api/recipes?_app_id=app-id&_app_key=app-key&your _search_parameters` to search recipes
- The parameter `allowedCuisine[]` filter the search results based a specific cuisine
- The parameters `salty`, `sour`, `sweet`, `bitter`, `meaty` and `piquant` specify the expected taste of the results
- The API returns JSON objects containing information about the recipes matching the query argument
  In the case where Yummly limits our usage of the API, we will build a web crawler based on the following pattern in the website’s HTML DOC and URLs:
- The search URLs are structured in the following way: `https://www.yummly.com/recipes?q=search_parameters`
- The search parameters passed in the URL search query are similar to the ones described in the API documentation
- The returned HTML contains links to recipes information pages nested in div tags having the same class attribute
- The recipes information pages have also the same pattern and can be web scrapped using the techniques studied in the course

# Task Distribution

- Ali Alami-Idrissi: Coming up with the algorithm, Crawling the data, Work on final presentation
- Ali Benlalah: Coding up the algorithm, Plotting graphs during data analysis, Work on final presentation
- Zakaria Fikrat: Writing up the report, Preliminary data analysis, Work on final presentation
