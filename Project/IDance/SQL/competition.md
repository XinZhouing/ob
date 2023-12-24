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