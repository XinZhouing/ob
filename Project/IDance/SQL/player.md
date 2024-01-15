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

```json
"id": 680347,
                "org_id": 3892,
                "class_id": 0,
                "name": "季伟涵",
                "pinyin": "jwh,jiweihan",
                "sex": 1,
                "birthday": "2016-09-20T00:00:00.000Z",
                "id_card": "210682201609200035",
                "id_card_type": "CN",
                "phone": "",
                "email": "",
                "city_name": "",
                "country": "CN",
                "certification": 1,
                "remark": "",
                "player_inbox_id": 0,
                "del": 0,
                "del_time": 0,
                "del_uid": 0,
                "create_time": "2023-05-21T13:58:55.000Z",
                "last_modify_time": "2023-05-21T13:59:19.000Z"
```

# 选手分类
```json
            {
                "id": 6470,
                "title": "2",
                "pid": 0,
                "org_id": 3892,
                "sort": 0,
                "create_time": "2024-01-08T11:38:47.000Z",
                "last_modify_time": "2024-01-08T11:38:47.000Z",
                "count": 0
            },
```