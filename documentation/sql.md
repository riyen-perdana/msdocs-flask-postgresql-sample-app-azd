#### Create Table Restaurant
`CREATE TABLE restaurant (id SERIAL PRIMARY KEY, name VARCHAR(100) NOT NULL, street_address VARCHAR(250) NOT NULL, description VARCHAR(250) NOT NULL);`


#### Create Table review
`CREATE TABLE review 
(
	id SERIAL PRIMARY KEY, 
	restaurant_id INT REFERENCES restaurant(id) ON DELETE CASCADE ON UPDATE CASCADE, 
	user_name VARCHAR(100), 
	rating INT CHECK(rating >=1 AND rating <=5), 
	review_text TEXT, review_date DATE
);`