- 选手表player
```sql
CREATE TABLE player (
    ID INT AUTO_INCREMENT PRIMARY KEY,
    Name VARCHAR(255) NOT NULL,
    NationRegion VARCHAR(255) NOT NULL,
    IdentityNumber VARCHAR(255) NOT NULL,
    DOB DATE,
    Gender ENUM('M','F'),
    ApprovalStatus ENUM('APPROVED', 'PENDING')
);
```
- 选手单位关联表 (Player_Unit)
```sql
CREATE TABLE Player_Unit (
    ID INT AUTO_INCREMENT PRIMARY KEY,
    PlayerID INT,
    UnitID INT,
    FOREIGN KEY (PlayerID) REFERENCES Players(ID),
    FOREIGN KEY (UnitID) REFERENCES Units(ID)
);
```