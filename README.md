# PL-SQL

## introduction

My task is to build a PL/SQL-based application to insert and update records in a video rental store database and generate some reports. 

The database consists of only the following essential tables.

•	CUSTOMER(CUSTOMER_ID, CUSTOMER_NAME, EMAIL, PHONE_NUMBER,
     REGISTRATION_DATE, EXPIRATION_DATE, LAST_UPDATE_DATE);

•	VIDEO(VIDEO_ID, VIDEO_NAME, FORMAT, PUBLISH_DATE,
  MAXIMUM_RENTAL_DAYS);

•	VIDEO_COPY(COPY_ID, VIDEO_ID*, COPY_STATUS, NOTE);

•	RENTAL_HISTORY (CUSTOMER_ID*, COPY_ID*, CHECKOUT_DATE,
   DUE_DATE, RETURN_DATE);


The primary keys are marked in red and the foreign keys are marked with asterisks.

VIDEO_COPY(COPY_STATUS): 	A – Available, R – Rented, D – Damaged

Each video in the VIDEO table has at least one video copy in the VIDEO_COPY table.

## part 1: Create table

Code for create 4 tables and import data to the tables
```
//(1)Customer table
CREATE TABLE customer 
( CUSTOMER_ID    		NUMBER 	 PRIMARY KEY,
  CUSTOMER_NAME      	VARCHAR2(30) NOT NULL,
  EMAIL   			VARCHAR2(50) NOT NULL,
  PHONE_NUMBER		VARCHAR2(15) NOT NULL,
  REGISTRATION_DATE	DATE NOT NULL,
  EXPIRATION_DATE  	DATE NOT NULL, 
  LAST_UPDATE_DATE	DATE NOT NULL);
/
INSERT INTO customer SELECT * FROM ####.customer; // #### is from other database and import to this database
COMMIT;
/
SELECT COUNT(*) FROM customer;


//(2) Video table
CREATE TABLE video 
( VIDEO_ID			NUMBER 		PRIMARY KEY,
  VIDEO_NAME     		VARCHAR2(100) 	NOT NULL,
  FORMAT			VARCHAR2(100) 	NOT NULL,
  PUBLISH_DATE  		DATE 			NOT NULL,
  MAXIMUM_RENTAL_DAYS	NUMBER(3) 		NOT NULL
); 
/
INSERT INTO video SELECT * FROM ####.video;
COMMIT;
/
SELECT COUNT(*) FROM video;

//(3) video copy
CREATE TABLE video_copy 
( COPY_ID       	NUMBER 	PRIMARY KEY,
  VIDEO_ID      	NUMBER 	NOT NULL REFERENCES VIDEO (VIDEO_ID),
  COPY_STATUS   	CHAR NOT NULL CONSTRAINT ck_item 
				CHECK (COPY_STATUS in ('A', 'R', 'D')),
  NOTE		VARCHAR2(200));
/
INSERT INTO video_copy SELECT * FROM ####.video_copy;
COMMIT;
/
SELECT COUNT(*) FROM video_copy;


// (4)rental history data
CREATE TABLE rental_history
( CUSTOMER_ID    	NUMBER 	REFERENCES CUSTOMER (CUSTOMER_ID),
  COPY_ID  		NUMBER 	REFERENCES VIDEO_COPY (COPY_ID),
  CHECKOUT_DATE   DATE NOT NULL,
  DUE_DATE  	DATE NOT NULL,
  RETURN_DATE  	DATE,
  NOTE		VARCHAR2(200),
  CONSTRAINT 	pk_rental PRIMARY KEY 
				(CUSTOMER_ID, COPY_ID, CHECKOUT_DATE));
/
INSERT INTO rental_history SELECT * FROM ####.rental_history;
COMMIT;
/
SELECT COUNT(*) FROM rental_history;
```
## Part 2
### (1)new_customer_registration()
* Create a procedure called new_customer_registration to add a new customer to the CUSTOMER table. 
```
date, 'yyyymmdd') < 
TO_CHAR(in_registration_date, 'yyyymmdd') THEN
          	DBMS_OUTPUT.PUT_LINE('Invalid expiration date!');
		RETURN;
	END IF;
  
	INSERT INTO customer 
       	VALUES(in_customer_id, UPPER(in_customer_name), in_email, 
			in_phone_number, in_registration_date, in_expiration_date,
sysdate);
	COMMIT;
	
      DBMS_OUTPUT.PUT_LINE
		(INITCAP(in_customer_name) || 
' has been added into the customer table.');
		
EXCEPTION
	WHEN OTHERS THEN
		DBMS_OUTPUT.PUT_LINE('My exception: ' || 
			TO_CHAR(SQLCODE) || '   ' || SQLERRM);
END;	

```
* Testing the procedure

```
•	EXEC new_customer_registration(2009, 'Adams', 'adams_1@yahoo.com', '3123621111', '02-OCT-2016', '01-OCT-2018')
Dbms Output: 	Adams has been added into the customer table.

•	EXEC new_customer_registration(2010, '', 'ford1@yahoo.com', 
'3123622222', '02-OCT-2016', '01-OCT-2018')
	Dbms Output:	Invalid customer name!
•	……
```
### (2)reset_expiration_date()
* Create a procedure called reset_expiration_date to reset an existing customer’s expiration date. 

```
CREATE OR REPLACE PROCEDURE reset_expiration_date 
(
in_customer_id 		NUMBER,
in_new_expiration_date 	DATE)
is 
v_n number;
begin
select count(*) into v_n from customer where customer_id=in_customer_id;
if v_n=0 then
dbms_output.put_line('Invalid customer ID');
return;
end if;

if in_new_expiration_date is null or to_char(in_new_expiration_date,'DD-MON-YYYY')<sysdate-1 then
dbms_output.put_line('invalid expiration date');
return;
end if;

update customer set 
expiration_date=in_new_expiration_date,
LAST_UPDATE_DATE=sysdate
where customer_id=in_customer_id;
dbms_output.put_line('The expiration date has been updated.');
EXCEPTION
	WHEN OTHERS THEN
		DBMS_OUTPUT.PUT_LINE('My exception: ' || 
			TO_CHAR(SQLCODE) || '   ' || SQLERRM);

end;
```
### (3)video_search()
* Create a procedure called video_search to search a video and display the name, copy ID, format, and status of the video’s copies. In addition, the checkout dates and due dates are also displayed for unreturned copies. The damaged copies (COPY_STATUS = 'D') are excluded in your output. Sort your output by the video name (NAME) and then the copy ID (COPY_ID).

* The video name is not case-sensitive (e.g., Ocean = ocean). The format of a video is not case-sensitive (e.g., DVD = dvd).
```
CREATE OR REPLACE PROCEDURE video_search 
(
in_video_name 	VARCHAR2, 
in_format 		VARCHAR2 DEFAULT NULL
)
is
cursor c_svideo is 
select video_name,c.copy_id,format,copy_status,max(checkout_date) as checkout_date,max(due_date) as due_date
from video_copy c
left outer join video v on v.video_id=c.video_id
left outer join rental_history h on c.copy_id=h.copy_id
WHERE UPPER(video_name) like '%' || UPPER(in_video_name) ||  '%'
 and UPPER(format) like '%' || UPPER(in_format) ||  '%' and copy_status!='D'
 group by video_name,c.copy_id,format,copy_status
order by video_name,c.copy_id;
v_status varchar2(30);
v_n number;
v_an number;
begin
 select count(*) into v_n from video v left outer join video_copy c on v.video_id=c.video_id WHERE UPPER(video_name) like '%' || UPPER(in_video_name) ||  '%'
 and UPPER(format) like '%' || UPPER(in_format) ||  '%'  and copy_status!='D' order by video_name;
 if v_n=0 then
 if in_format is not null then
  dbms_output.put_line('*****'||v_n||' results found for '||upper(in_video_name)||'('||in_format||')'||'. *****');
  else
  dbms_output.put_line('*****'||v_n||' results found for '||upper(in_video_name)||'. *****');
 end if;
 else
 select count(*) into v_an from video v left outer join video_copy c on v.video_id=c.video_id where UPPER(video_name) like '%' || UPPER(in_video_name) ||  '%' and UPPER(format) like '%' || UPPER(in_format) ||  '%' and Copy_Status='A';
if in_format is null then
dbms_output.put_line('***** '||v_n||' results found for '||upper(in_video_name)||'. (Available copies: '||v_an||')'||' *****');
else
dbms_output.put_line('***** '||v_n||' results found for '||upper(in_video_name)||'('||upper(in_format)||')'||'. (Available copies: '||v_an||')'||' *****');
end if;
 dbms_output.put_line(rpad(' ', 80,' '));
 dbms_output.put_line(rpad('VIDEO NAME',20)||RPAD('COPY ID',10)||RPAD('FORMAT',10)||RPAD('COPY STATUS',15)||RPAD('CHECKOUT DATE',20)||RPAD('DUE DATE',20));
  dbms_output.put_line(lpad(' ',100,'-'));

for idx in c_svideo loop
if idx.copy_status='R'then
v_status:='Rented';
dbms_output.put_line(rpad(idx.VIDEO_NAME,20)||RPAD(idx.COPY_ID,10)||RPAD(idx.FORMAT,10)||RPAD(v_STATUS,15)||RPAD(to_char(idx.CHECKOUT_DATE,'DD-MON-YYYY'),20)||RPAD(to_char(idx.DUE_DATE,'DD-MON-YYYY'),20));
else
v_status:='Available';

dbms_output.put_line(rpad(idx.VIDEO_NAME,20)||RPAD(idx.COPY_ID,10)||RPAD(idx.FORMAT,10)||RPAD(v_STATUS,15));
end if;
end loop;
end if;
end;
```

* Testing the procedure

```
•	EXEC video_search('ocean')
	Dbms Output: 

	***** 0 results found for ocean. *****

•	EXEC video_search('PRETTY WOMAN', 'Blu-Ray')
	Dbms Output:

	***** 0 results found for PRETTY WOMAN (Blu-Ray). *****

•	EXEC video_search('Pretty Woman')
	Dbms Output:

	***** 3 results found for Pretty Woman. (Available copies: 3) *****

 
	VIDEO NAME           	 COPY ID    FORMAT      COPY STATUS    CHECKOUT DATE       DUE DATE
	---------------------------------------------------------------------------------------------
	PRETTY WOMAN                  6000    VHS TAPE    Available
	PRETTY WOMAN                  6001    VHS TAPE    Available
	PRETTY WOMAN                  6015    DVD         Available

•	EXEC video_search('Another')
	Dbms Output:

       ***** 4 results found for Another. (Available copies: 2) *****
 
	VIDEO NAME           	 COPY ID    FORMAT      COPY STATUS    CHECKOUT DATE       DUE DATE
	---------------------------------------------------------------------------------------------
	DIE ANOTHER DAY               6010    VHS TAPE    Available
	DIE ANOTHER DAY               6011    VHS TAPE    Rented           20-APR-2016    04-MAY-2016
	DIE ANOTHER DAY               6014    DVD         Available
	DIE ANOTHER DAY               6016    BLU-RAY     Rented           01-OCT-2016    04-OCT-2016

•	EXEC video_search('ANOTHER', 'Dvd')
	Dbms Output:

	***** 1 result found for ANOTHER (Dvd). (Available copies: 1) *****
 
	VIDEO NAME           	 COPY ID    FORMAT      COPY STATUS    CHECKOUT DATE       DUE DATE
	---------------------------------------------------------------------------------------------
	DIE ANOTHER DAY               6014    DVD         Available

•	EXEC video_search('Story')
	Dbms Output:

	***** 7 results found for Story. (Available copies: 4) *****
 
	VIDEO NAME           	 COPY ID    FORMAT      COPY STATUS    CHECKOUT DATE       DUE DATE
	---------------------------------------------------------------------------------------------
	TOY STORY                     6002    VHS TAPE    Rented           09-APR-2016    23-APR-2016
	TOY STORY                     6003    VHS TAPE    Available
	TOY STORY                     6017    DVD         Rented           01-OCT-2016    08-OCT-2016
	TOY STORY 2                   6009    VHS TAPE    Available
	TOY STORY 2                   6018    DVD         Rented           28-APR-2016    05-MAY-2016
	TOY STORY 2                   6019    DVD         Available
	TOY STORY 2                   6020    BLU-RAY     Available
```

### (4)video_checkout()
* Create a procedure called video_checkout to record a new rental. When the video is successfully checked out, you need to insert a new row (record) into the RENTAL_HISTORY table and update the corresponding row (record) in the VIDEO_COPY table. Otherwise, the action is denied. 
```
CREATE OR REPLACE PROCEDURE video_checkout
(	
in_customer_id		NUMBER, 
	in_copy_id 			NUMBER, 
	in_video_checkout_date 	DATE 
)
is
v_maxrent Video.Maximum_Rental_Days%type;
v_video video_copy.video_id%type;
v_n number;
v_due date;
v_expire customer.expiration_date%type;
v_status video_copy.copy_status%type;
v_return_date rental_history.return_date%type;

cursor c_video_id is
select video_id, c.copy_id from video_copy c
left outer join rental_history r
on c.copy_id=r.copy_id
where video_id= v_video and customer_id=in_customer_id and return_date is null
order by video_id; 

cursor c_same is
select copy_id, customer_id, checkout_date from rental_history order by customer_id;

begin
select count(*) into v_n from customer where customer_id=in_customer_id;

if v_n=0 then
dbms_output.put_line('invalid customer id');
return;
end if;

select count(*) into v_n from Video_Copy where copy_id=in_copy_id ;

if v_n=0 then
dbms_output.put_line('invalid copy id');
return;
end if;

select count(*) into v_n from video_copy where copy_id=in_copy_id and copy_status='A';

if v_n=0 then
dbms_output.put_line('copy is not available');
return;
end if;

select expiration_date into v_expire from customer where customer_id=in_customer_id;
if v_expire<sysdate then
dbms_output.put_line('expiration customer id');
return;
end if;


if in_video_checkout_date> sysdate or in_video_checkout_date is null then
dbms_output.put_line('invalid checkout date');
return;
end if;

select count(*) into v_n from rental_history r
left outer join video_copy c on r.copy_id=c.copy_id
where customer_id=in_customer_id and copy_status='R';

if v_n>8 then
dbms_output.put_line('cannot rent a copy before return one of eight copies you checked out');
return;
end if;

select video_id into v_video from video_copy where copy_id=in_copy_id;
for idx in c_video_id loop
if idx.video_id=v_video then
dbms_output.put_Line('you should return the video with same name which you have.');
return;
end if;
end loop;

for idx in c_same loop
if idx.customer_id=in_customer_id and idx.copy_id=in_copy_id and idx.checkout_date=in_video_checkout_date then
dbms_output.put_Line('Same customer id cannot rent same video(same copy id) on same day repeatly after rented it');
end if;
end loop;

select Maximum_Rental_Days into v_maxrent from video where video_id=v_video;
v_due:=in_video_checkout_date+v_maxrent;
insert into rental_history values(in_customer_id, in_copy_id, in_video_checkout_date,v_due,null,null);
update video_copy set
copy_status='R' where copy_id=in_copy_id;
dbms_output.put_Line('Rented');
end;
```

### (5)video_return()
```
CREATE OR REPLACE PROCEDURE video_return
(
in_copy_id 			NUMBER, 
in_video_return_date 	DATE
)
is
v_n number;
v_status video_copy.copy_status%type;
v_checkout_date rental_history.checkout_date%type;
v_due rental_history.due_date%type;
begin

select count(*) into v_n from video_copy where copy_id=in_copy_id ;
if v_n=0 then
dbms_output.put_line('invalid copy id! ');
return;
end if;


select count(*) into v_n from video_copy where copy_id=in_copy_id and copy_status!='R';
if v_n=1 then
dbms_output.put_line('This video copy has not been rented.');
return;
end if;

select checkout_date into v_checkout_date from rental_history where copy_id=in_copy_id and return_date is null;
if in_video_return_date>sysdate or in_video_return_date<v_checkout_date then
dbms_output.put_line('Invalid Date');
return;
end if;

if in_video_return_date >v_due then
dbms_output.put_line('beyond due date');
end if;

update video_copy set
copy_status='A' where copy_id=in_copy_id;

update rental_history set
return_date=in_video_return_date where copy_id=in_copy_id and return_date is null;
dbms_output.put_line('returned');

end;
```
### (6)print_unreturned_video()
```
CREATE OR REPLACE PROCEDURE print_unreturned_video
(
in_customer_id NUMBER
)

is

cursor c_customer is
select customer_id, customer_name, email,phone_number,EXPIRATION_DATE
from customer
where customer_id=in_customer_id;

cursor c_unreturn (p_did number) is
select c.copy_id,video_name,format,checkout_date,due_date
from video_copy c
left outer join video o 
on c.video_id=o.video_id
left outer join rental_history r
on c.copy_id=r.copy_id
where r.customer_id=p_did and return_date is null
order by due_date,video_name;

v_n number;
v_count number;
nothing varchar2(10);
max_checkout rental_history.checkout_date%type;
min_checkout rental_history.checkout_date%type;
begin
select count(*) into v_n from customer where customer_id=in_customer_id;

if v_n=0 then
dbms_output.put_line('The customer (id='||in_customer_id||') is not in the customer table. ');
return;
else

for idx_cust in c_customer loop

select max(checkout_date) into max_checkout from rental_history where customer_id= in_customer_id;
select min(checkout_date) into min_checkout from rental_history where customer_id= in_customer_id;
select count(*) into v_count from rental_history where customer_id=in_customer_id and return_date is null;

dbms_output.put_line(Lpad(' ',55,'-'));
dbms_output.put_line(rpad('Customer ID: ',30)||idx_cust.customer_id);
dbms_output.put_line(rpad('Customer Name: ',30)||idx_cust.customer_name);
dbms_output.put_line(rpad('Email: ',30)||idx_cust.email);
dbms_output.put_line(rpad('Phone Number: ',30)||idx_cust.phone_number);
dbms_output.put_line(rpad('Expiration Date: ',30)||TO_CHAR(idx_cust.expiration_date,'DD-MON-YYYY'));
if min_checkout is null then
dbms_output.put_line(rpad('First Checkout Date: ',30)||'N/A');
else
dbms_output.put_line(rpad('First Checkout Date: ',30)||TO_CHAR(min_checkout,'DD-MON-YYYY'));
end if;
if max_checkout is null then
dbms_output.put_line(rpad('Last Checkout Date: ',30)||'N/A');
else
dbms_output.put_line(rpad('Last Checkout Date: ',30)||TO_CHAR(max_checkout,'DD-MON-YYYY'));
end if;
dbms_output.put_line(Lpad(' ',55,'-'));
dbms_output.put_line('Number of Unreturned Videos: '||v_count);

if v_count=0 then
dbms_output.put_line(Lpad(' ',55,'-'));
else
dbms_output.put_line(Lpad(' ',55,'-'));
for idx in c_unreturn(idx_cust.customer_id) loop
dbms_output.put_line(rpad('Copy ID: ',30)||idx.copy_id);
dbms_output.put_line(rpad('Title: ',30)||idx.video_name);
dbms_output.put_line(rpad('Fromat: ',30)||idx.format);
dbms_output.put_line(rpad('Checkout Date: ',30)||to_char(idx.checkout_date,'DD-MON-YYYY'));
dbms_output.put_line(rpad('Due Date: ',30)||to_char(idx.due_date,'DD-MON-YYYY'));
dbms_output.put_line(Lpad(' ',55,'-'));

end loop;

end if;
end loop;
end if;
end;
```

* Testing the procedure

```

•	EXEC print_unreturned_video(90)

	Dbms Output: 
	The customer (id = 90) is not in the customer table.

•	EXEC print_unreturned_video(2004)

Dbms Output: 
----------------------------------------
Customer ID:             2004
Customer Name:           JOHNSON
Email:                   jo12@gmail.com
Phone Number:            7732015678
Expiration Date:         21-APR-2017
First Checkout Date:     01-MAY-2015
Last Checkout Date:      01-MAY-2015
----------------------------------------
Number of Unreturned Videos:  0
----------------------------------------

•	EXEC print_unreturned_video(2008)

Dbms Output: 
----------------------------------------
Customer ID:             2008
Customer Name:           SCOTT
Email:                   scott@hotmail.com
Phone Number:            7701239876
Expiration Date:         30-DEC-2011
First Checkout Date:     N/A
Last Checkout Date:      N/A
----------------------------------------
Number of Unreturned Videos:  0
----------------------------------------

•	EXEC print_unreturned_video(2002)

Dbms Output: 
----------------------------------------
Customer ID:             2002
Customer Name:           JONES
Email:                   jones@hotmail.com
Phone Number:            6302077890
Expiration Date:         02-MAR-2017
First Checkout Date:     04-MAR-2015
Last Checkout Date:      01-OCT-2016
----------------------------------------
Number of Unreturned Videos:  3
----------------------------------------
Copy ID:         6007
Title:           TARZAN
Format:          VHS TAPE
Checkout Date:   22-APR-2016
Due Date:        06-MAY-2016
----------------------------------------
Copy ID:         6016
Title:           DIE ANOTHER DAY
Format:          BLU-RAY
Checkout Date:   01-OCT-2016
Due Date:        04-OCT-2016
----------------------------------------
Copy ID:         6017
Title:           TOY STORY
Format:          DVD
Checkout Date:   01-OCT-2016
Due Date:        08-OCT-2016
----------------------------------------
```
### (7)Package video_rental_pkg
* Group all the above subprograms (new_ustomer_registration, reset_expiration_date, video_search, video_checkout …) together in a package (package specification and package body) called video_rental_pkg. 
```
CREATE OR REPLACE PACKAGE video_rental_pkg AS
PROCEDURE new_customer_registration(	in_customer_id NUMBER,in_customer_name VARCHAR2,
in_email VARCHAR2, in_phone_number VARCHAR2,in_registration_date	DATE,in_expiration_date	DATE);
PROCEDURE reset_expiration_date (in_customer_id 		NUMBER,in_new_expiration_date 	DATE);
PROCEDURE video_search (in_video_name VARCHAR2, in_format VARCHAR2 DEFAULT NULL);
PROCEDURE video_checkout(	in_customer_id		NUMBER, in_copy_id 			NUMBER, in_video_checkout_date 	DATE);
PROCEDURE video_return(in_copy_id 			NUMBER, in_video_return_date 	DATE);
PROCEDURE print_unreturned_video(in_customer_id NUMBER);
END video_rental_pkg;
/

CREATE OR REPLACE PACKAGE BODY video_rental_pkg AS

PROCEDURE new_customer_registration
(	
	in_customer_id		NUMBER,
	in_customer_name 		VARCHAR2,
in_email 			VARCHAR2, 
in_phone_number 		VARCHAR2,
 	in_registration_date	DATE,
	in_expiration_date	DATE) 
IS
	v_count		NUMBER;
BEGIN
	IF in_customer_id <= 0 THEN
		DBMS_OUTPUT.PUT_LINE('Invalid customer ID!');
		RETURN;
	END IF;

	SELECT 	COUNT(*)
	INTO		v_count
	FROM		customer
	WHERE		customer_id = in_customer_id;

	IF v_count != 0 THEN
		DBMS_OUTPUT.PUT_LINE('Invalid customer ID!');
		RETURN;
	END IF;

  	IF in_customer_name is NULL THEN 
          	DBMS_OUTPUT.PUT_LINE('Invalid customer name!');
		RETURN;
	END IF;

     	IF in_email is NULL THEN 
          	DBMS_OUTPUT.PUT_LINE('Invalid email address!');
		RETURN;
	END IF;

    	IF in_registration_date IS NULL OR 
			TO_CHAR(in_registration_date, 'yyyymmdd') > 
TO_CHAR(sysdate, 'yyyymmdd') THEN
          	DBMS_OUTPUT.PUT_LINE('Invalid registration date!');
		RETURN;
	END IF;

IF in_expiration_date IS NULL OR
		TO_CHAR(in_expiration_date, 'yyyymmdd') < 
TO_CHAR(in_registration_date, 'yyyymmdd') THEN
          	DBMS_OUTPUT.PUT_LINE('Invalid expiration date!');
		RETURN;
	END IF;
  
	INSERT INTO customer 
       	VALUES(in_customer_id, UPPER(in_customer_name), in_email, 
			in_phone_number, in_registration_date, in_expiration_date,
sysdate);
	COMMIT;
	
      DBMS_OUTPUT.PUT_LINE
		(INITCAP(in_customer_name) || 
' has been added into the customer table.');
		
EXCEPTION
	WHEN OTHERS THEN
		DBMS_OUTPUT.PUT_LINE('My exception: ' || 
			TO_CHAR(SQLCODE) || '   ' || SQLERRM);
END;

PROCEDURE reset_expiration_date 
(
in_customer_id 		NUMBER,
in_new_expiration_date 	DATE)
is 
v_n number;
begin
select count(*) into v_n from customer where customer_id=in_customer_id;
if v_n=0 then
dbms_output.put_line('Invalid customer ID');
return;
end if;

if in_new_expiration_date is null or to_char(in_new_expiration_date,'DD-MON-YYYY')<sysdate-1 then
dbms_output.put_line('invalid expiration date');
return;
end if;
update customer set 
expiration_date=in_new_expiration_date,
LAST_UPDATE_DATE=sysdate
where customer_id=in_customer_id;
dbms_output.put_line('The expiration date has been updated.');
EXCEPTION
	WHEN OTHERS THEN
		DBMS_OUTPUT.PUT_LINE('My exception: ' || 
			TO_CHAR(SQLCODE) || '   ' || SQLERRM);


end;

PROCEDURE video_search 
(
in_video_name 	VARCHAR2, 
in_format 		VARCHAR2 DEFAULT NULL
)
is
cursor c_svideo is 
select video_name,c.copy_id,format,copy_status,max(checkout_date) as checkout_date,max(due_date) as due_date
from video_copy c
left outer join video v on v.video_id=c.video_id
left outer join rental_history h on c.copy_id=h.copy_id
WHERE UPPER(video_name) like '%' || UPPER(in_video_name) ||  '%'
 and UPPER(format) like '%' || UPPER(in_format) ||  '%' and copy_status!='D'
 group by video_name,c.copy_id,format,copy_status
order by video_name,c.copy_id;
v_status varchar2(30);
v_n number;
v_an number;
begin
 select count(*) into v_n from video v left outer join video_copy c on v.video_id=c.video_id WHERE UPPER(video_name) like '%' || UPPER(in_video_name) ||  '%'
 and UPPER(format) like '%' || UPPER(in_format) ||  '%'  and copy_status!='D' order by video_name;
 if v_n=0 then
 if in_format is not null then
  dbms_output.put_line('*****'||v_n||' results found for '||upper(in_video_name)||'('||in_format||')'||'. *****');
  else
  dbms_output.put_line('*****'||v_n||' results found for '||upper(in_video_name)||'. *****');
 end if;
 else
 select count(*) into v_an from video v left outer join video_copy c on v.video_id=c.video_id where UPPER(video_name) like '%' || UPPER(in_video_name) ||  '%' and UPPER(format) like '%' || UPPER(in_format) ||  '%' and Copy_Status='A';
if in_format is null then
dbms_output.put_line('***** '||v_n||' results found for '||upper(in_video_name)||'. (Available copies: '||v_an||')'||' *****');
else
dbms_output.put_line('***** '||v_n||' results found for '||upper(in_video_name)||'('||upper(in_format)||')'||'. (Available copies: '||v_an||')'||' *****');
end if;
 dbms_output.put_line(rpad(' ', 80,' '));
 dbms_output.put_line(rpad('VIDEO NAME',20)||RPAD('COPY ID',10)||RPAD('FORMAT',10)||RPAD('COPY STATUS',15)||RPAD('CHECKOUT DATE',20)||RPAD('DUE DATE',20));
  dbms_output.put_line(lpad(' ',100,'-'));

for idx in c_svideo loop
if idx.copy_status='R'then
v_status:='Rented';
dbms_output.put_line(rpad(idx.VIDEO_NAME,20)||RPAD(idx.COPY_ID,10)||RPAD(idx.FORMAT,10)||RPAD(v_STATUS,15)||RPAD(to_char(idx.CHECKOUT_DATE,'DD-MON-YYYY'),20)||RPAD(to_char(idx.DUE_DATE,'DD-MON-YYYY'),20));
else
v_status:='Available';

dbms_output.put_line(rpad(idx.VIDEO_NAME,20)||RPAD(idx.COPY_ID,10)||RPAD(idx.FORMAT,10)||RPAD(v_STATUS,15));
end if;
end loop;
end if;
end;

PROCEDURE video_checkout
(	
in_customer_id		NUMBER, 
	in_copy_id 			NUMBER, 
	in_video_checkout_date 	DATE 
)
is
v_maxrent Video.Maximum_Rental_Days%type;
v_video video_copy.video_id%type;
v_n number;
v_due date;
v_expire customer.expiration_date%type;
v_status video_copy.copy_status%type;
v_return_date rental_history.return_date%type;

cursor c_video_id is
select video_id, c.copy_id from video_copy c
left outer join rental_history r
on c.copy_id=r.copy_id
where video_id= v_video and customer_id=in_customer_id and return_date is null
order by video_id; 

cursor c_same is
select copy_id, customer_id, checkout_date from rental_history order by customer_id;

begin
select count(*) into v_n from customer where customer_id=in_customer_id;

if v_n=0 then
dbms_output.put_line('invalid customer id');
return;
end if;

select count(*) into v_n from Video_Copy where copy_id=in_copy_id ;

if v_n=0 then
dbms_output.put_line('invalid copy id');
return;
end if;

select count(*) into v_n from video_copy where copy_id=in_copy_id and copy_status='A';

if v_n=0 then
dbms_output.put_line('copy is not available');
return;
end if;

select expiration_date into v_expire from customer where customer_id=in_customer_id;
if v_expire<sysdate then
dbms_output.put_line('expiration customer id');
return;
end if;


if in_video_checkout_date> sysdate or in_video_checkout_date is null then
dbms_output.put_line('invalid checkout date');
return;
end if;

select count(*) into v_n from rental_history r
left outer join video_copy c on r.copy_id=c.copy_id
where customer_id=in_customer_id and copy_status='R';

if v_n>8 then
dbms_output.put_line('cannot rent a copy before return one of eight copies you checked out');
return;
end if;

select video_id into v_video from video_copy where copy_id=in_copy_id;
for idx in c_video_id loop
if idx.video_id=v_video then
dbms_output.put_Line('you should return the video with same name which you have.');
return;
end if;
end loop;

for idx in c_same loop
if idx.customer_id=in_customer_id and idx.copy_id=in_copy_id and idx.checkout_date=in_video_checkout_date then
dbms_output.put_Line('Same customer id cannot rent same video(same copy id) on same day repeatly after rented it');
end if;
end loop;

select Maximum_Rental_Days into v_maxrent from video where video_id=v_video;
v_due:=in_video_checkout_date+v_maxrent;
insert into rental_history values(in_customer_id, in_copy_id, in_video_checkout_date,v_due,null,null);
update video_copy set
copy_status='R' where copy_id=in_copy_id;
dbms_output.put_Line('Rented');
end;

PROCEDURE video_return
(
in_copy_id 			NUMBER, 
in_video_return_date 	DATE
)
is
v_n number;
v_status video_copy.copy_status%type;
v_checkout_date rental_history.checkout_date%type;
v_due rental_history.due_date%type;
begin

select count(*) into v_n from video_copy where copy_id=in_copy_id ;
if v_n=0 then
dbms_output.put_line('invalid copy id! ');
return;
end if;


select count(*) into v_n from video_copy where copy_id=in_copy_id and copy_status!='R';
if v_n=1 then
dbms_output.put_line('This video copy has not been rented.');
return;
end if;

select checkout_date into v_checkout_date from rental_history where copy_id=in_copy_id and return_date is null;
if in_video_return_date>sysdate or in_video_return_date<v_checkout_date then
dbms_output.put_line('Invalid Date');
return;
end if;

if in_video_return_date >v_due then
dbms_output.put_line('beyond due date');
end if;

update video_copy set
copy_status='A' where copy_id=in_copy_id;

update rental_history set
return_date=in_video_return_date where copy_id=in_copy_id and return_date is null;
dbms_output.put_line('returned');

end;


PROCEDURE print_unreturned_video
(
in_customer_id NUMBER
)

is

cursor c_customer is
select customer_id, customer_name, email,phone_number,EXPIRATION_DATE
from customer
where customer_id=in_customer_id;

cursor c_unreturn (p_did number) is
select c.copy_id,video_name,format,checkout_date,due_date
from video_copy c
left outer join video o 
on c.video_id=o.video_id
left outer join rental_history r
on c.copy_id=r.copy_id
where r.customer_id=p_did and return_date is null
order by due_date,video_name;

v_n number;
v_count number;
nothing varchar2(10);
max_checkout rental_history.checkout_date%type;
min_checkout rental_history.checkout_date%type;
begin
select count(*) into v_n from customer where customer_id=in_customer_id;

if v_n=0 then
dbms_output.put_line('The customer (id='||in_customer_id||') is not in the customer table. ');
return;
else

for idx_cust in c_customer loop

select max(checkout_date) into max_checkout from rental_history where customer_id= in_customer_id;
select min(checkout_date) into min_checkout from rental_history where customer_id= in_customer_id;
select count(*) into v_count from rental_history where customer_id=in_customer_id and return_date is null;

dbms_output.put_line(Lpad(' ',55,'-'));
dbms_output.put_line(rpad('Customer ID: ',30)||idx_cust.customer_id);
dbms_output.put_line(rpad('Customer Name: ',30)||idx_cust.customer_name);
dbms_output.put_line(rpad('Email: ',30)||idx_cust.email);
dbms_output.put_line(rpad('Phone Number: ',30)||idx_cust.phone_number);
dbms_output.put_line(rpad('Expiration Date: ',30)||TO_CHAR(idx_cust.expiration_date,'DD-MON-YYYY'));
if min_checkout is null then
dbms_output.put_line(rpad('First Checkout Date: ',30)||'N/A');
else
dbms_output.put_line(rpad('First Checkout Date: ',30)||TO_CHAR(min_checkout,'DD-MON-YYYY'));
end if;
if max_checkout is null then
dbms_output.put_line(rpad('Last Checkout Date: ',30)||'N/A');
else
dbms_output.put_line(rpad('Last Checkout Date: ',30)||TO_CHAR(max_checkout,'DD-MON-YYYY'));
end if;
dbms_output.put_line(Lpad(' ',55,'-'));
dbms_output.put_line('Number of Unreturned Videos: '||v_count);

if v_count=0 then
dbms_output.put_line(Lpad(' ',55,'-'));
else
dbms_output.put_line(Lpad(' ',55,'-'));
for idx in c_unreturn(idx_cust.customer_id) loop
dbms_output.put_line(rpad('Copy ID: ',30)||idx.copy_id);
dbms_output.put_line(rpad('Title: ',30)||idx.video_name);
dbms_output.put_line(rpad('Fromat: ',30)||idx.format);
dbms_output.put_line(rpad('Checkout Date: ',30)||to_char(idx.checkout_date,'DD-MON-YYYY'));
dbms_output.put_line(rpad('Due Date: ',30)||to_char(idx.due_date,'DD-MON-YYYY'));
dbms_output.put_line(Lpad(' ',55,'-'));

end loop;

end if;
end loop;
end if;
end;


END video_rental_pkg;








test:
---------------------------------------------------------------------------------------------------------------------------------
begin
--video_rental_pkg.new_customer_registration(2011, 'QI', 'ford1@yahoo.com','3123622222', '02-OCT-2016', '01-OCT-2018');
--video_rental_pkg.reset_expiration_date('2011','20-Nov-2016');
--video_rental_pkg.video_search('tit');
--video_rental_pkg.video_checkout('2010','6004','19-NOV-2016');
--video_rental_pkg.video_return('6013','20-NOV-2015');
video_rental_pkg.print_unreturned_video(2002);
end;
```
