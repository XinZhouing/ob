```sql
CREATE TABLE units (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  leader_name VARCHAR(50) NOT NULL,
  leader_contact VARCHAR(50) NOT NULL
);

CREATE TABLE unit_members (
  id INT PRIMARY KEY AUTO_INCREMENT,
  unit_id INT,
  user_id INT,
  FOREIGN KEY (unit_id) REFERENCES units(id),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  contact VARCHAR(50) NOT NULL
);
```
