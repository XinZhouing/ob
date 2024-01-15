- 赛事表
```sql
CREATE TABLE Event ( 
	id INT PRIMARY KEY,
	title VARCHAR(100) COMMENT'赛事名称', 
	status VARCHAR(20) COMMENT'赛事状态，（可报名、直播中）',
	event_time VARCHAR(50) COMMENT'赛事举行日期，可能包括开始和结束日期',
	enroll_deadline DATE COMMENT'报名截止日期',
	organizer VARCHAR(100) COMMENT'主办方', 
	host_city VARCHAR(50) COMMENT'主办城市，如：重庆，武汉',
	venue VARCHAR(100) COMMENT'赛事场地',
	contact_number VARCHAR(20) COMMENT'联系电话',
	available_services VARCHAR(200) COMMENT'提供的服务，如：酒店，门票'
);
```

```json
{
        "id": 735,
        "logo": "",
        "banner": "image/18/183b49938368c5a49eec022e24601522.png",
        "name": "2022中国·沈阳冠军舞者国际标准舞公开赛",
        "layout_lock": 1,
        "layout_complete": 1,
		"organizer_name": "沈阳市国际标准舞协会",
        "organizer_id": 20805,
        "location_name": "沈阳市碧桂园玛丽蒂姆酒店",
        "location_city": "沈阳市",
        "location_lon": 123.342607,
        "location_lat": 41.753101,
        "enroll_hide": 0,
        "enroll_disable": 0,
        "status": 0,
        "enroll_time_start": 1658215167144,
        "enroll_time_end": 1659974399144,
        "match_time_start": 1660807129050,
        "match_time_end": 1660807129050,
        "enroll_contact": "APP报名负责人：17576056676\n获取邀请码：17753536336 \n(主办方电话请在左下角规程按钮中查看)",
        "enroll_invite_type": 1,
        "enroll_invite_allow_null": 1,
        "enroll_verify_type": 1,
        "enroll_notice_mail": "",
        "enroll_notice_phone": "13354211110",
        "enroll_proclamation": "",
        "enroll_age_force": 0,
        "enroll_age_datum": 1660720724998,
        "inquiry_contact": "",
        "inquiry_proclamation": "",
        "thirdparty": 0,
        "thirdparty_url": "",
        "thirdparty_score_url": "",
        "rules": ""
    }
```