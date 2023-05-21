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
                              principal_name => 'FIN',
                              principal_type => xs_acl.ptype_db)); 
END;
/
```
