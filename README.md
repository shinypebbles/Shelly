# Shelly
Collect and process data from Shelly in an Oracle database

This project collects data from the [Shelly Plus Plug S](https://www.shelly.cloud/en/products/shop/shelly-plus-plug-s) and stores it in an Oracle database (v12.1). The data is displayed in a graph using Oracle APEX (v20.1).

The Shelly Plug delivers data in JSON, so let's create a data model to store the data.

```
CREATE TABLE  "PLUG" 
   (	"SEQ#" NUMBER NOT NULL ENABLE, 
	"HOSTNAME" VARCHAR2(40) NOT NULL ENABLE, 
	"NAME" VARCHAR2(40), 
	 CONSTRAINT "PLUG_PK" PRIMARY KEY ("SEQ#")
  USING INDEX  ENABLE
   )
/

CREATE TABLE  "PLUGSTATUS" 
   (	"PLUGSEQ#" NUMBER, 
	"DATA" CLOB, 
	 CONSTRAINT "PLUGSTATUS_JSON_CHK" CHECK (data IS JSON) ENABLE
   )
/
ALTER TABLE  "PLUGSTATUS" ADD CONSTRAINT "PLUGSTATUS_FK1" FOREIGN KEY ("PLUGSEQ#")
	  REFERENCES  "PLUG" ("SEQ#") ENABLE
/
```

The parent table contains the hostname of the Shelly plugs. The child table contains the collected data.

The data is collected through a REST API call. First we need to allow access to hosts in the network.

```
BEGIN
  DBMS_NETWORK_ACL_ADMIN.append_host_ace (
    host       => '*', 
    lower_port => 80,
    upper_port => 80,
    ace        => xs$ace_type(privilege_list => xs$name_list('http'),
                              principal_name => 'SHELLY',
                              principal_type => xs_acl.ptype_db)); 
END;
/
```
The following procedure fetches the [data](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Switch/#status) from the plug and inserts it in the database.
```
create or replace PROCEDURE getplugstatus AS
  req   UTL_HTTP.REQ;
  resp  UTL_HTTP.RESP;
  value VARCHAR2(1024);
BEGIN
  for c1_rec in (select seq# 
    , hostname
    from plug
    order by 1)
  LOOP
    BEGIN
      req := UTL_HTTP.BEGIN_REQUEST('http://'||c1_rec.hostname||'/rpc/Switch.GetStatus?id=0');
      UTL_HTTP.SET_HEADER(req, 'User-Agent', 'Mozilla/4.0');
      resp := UTL_HTTP.GET_RESPONSE(req);
      LOOP
        UTL_HTTP.READ_LINE(resp, value, TRUE);
        DBMS_OUTPUT.PUT_LINE(value);
        insert into plugstatus(plugseq#, data) values (c1_rec.seq#, value);
      END LOOP;
      UTL_HTTP.END_RESPONSE(resp);
    EXCEPTION
      WHEN UTL_HTTP.END_OF_BODY THEN
        UTL_HTTP.END_RESPONSE(resp);
      WHEN others THEN
        DBMS_OUTPUT.PUT_LINE('other error');
    END;
  END LOOP;
END;
```
Create a job to collect the data every minute
```
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
            job_name => '"SHELLY"."JOB_GET_STATUS"',
            job_type => 'PLSQL_BLOCK',
            job_action => 'begin getplugstatus; end;',
            number_of_arguments => 0,
            start_date => TO_TIMESTAMP_TZ('2015-11-17 21:35:00.000000000 EUROPE/AMSTERDAM','YYYY-MM-DD HH24:MI:SS.FF TZR'),
            repeat_interval => 'FREQ=MINUTELY',
            end_date => NULL,
            enabled => FALSE,
            auto_drop => TRUE,
            comments => 'Get Shelly status every minute');
 
    DBMS_SCHEDULER.SET_ATTRIBUTE( 
             name => '"SHELLY"."JOB_GET_STATUS"', 
             attribute => 'store_output', value => TRUE);
    DBMS_SCHEDULER.SET_ATTRIBUTE( 
             name => '"SHELLY"."JOB_GET_STATUS"', 
             attribute => 'logging_level', value => DBMS_SCHEDULER.LOGGING_OFF);
      
    DBMS_SCHEDULER.enable(
             name => '"SHELLY"."JOB_GET_STATUS"');
END;
```
The data needed for the graph is selected from the JSON data in the child table.
```
select to_number(S.data.apower) as power
, TO_DATE( '1970-01-01', 'YYYY-MM-DD' ) + NUMTODSINTERVAL( S.data.aenergy.minute_ts+7200, 'SECOND') as date_time, P.seq#
from plug P, plugstatus S
where P.seq#=S.plugseq#
  and P.seq#=:P88_SEQ
  and trunc(TO_DATE( '1970-01-01', 'YYYY-MM-DD' ) + NUMTODSINTERVAL( S.data.aenergy.minute_ts+7200, 'SECOND'))=trunc(to_date(:P88_DATUM)) order by  TO_DATE( '1970-01-01', 'YYYY-MM-DD' ) + NUMTODSINTERVAL( S.data.aenergy.minute_ts+7200, 'SECOND')
```
As the date is in UTC it is adjusted for CET by adding 7200 to it, this is not a good solution, it should be replaced by a more robust solution.
![Freezer power consumption](https://github.com/shinypebbles/Shelly/blob/main/Freezer.png)
