#patient表结构理解
- PDCDM字段对应
- 设计要点
- 本表数据处理需求点
- 本表数据存在的问题
-  数据处理要点
- 数据增量抽取后DDL语句
- 数据增量抽取后DDL语句
##PDCDM字段对应
| PDCDM扩展模型:     |                                                                 |            |        |          |             |                                                                          |
|--------------------|-----------------------------------------------------------------|------------|--------|----------|-------------|--------------------------------------------------------------------------|
|                    |                                                                 |            |        |          |             |                                                                          |
| Patient            | Patient表记录了患者的个人信息。包括出生年月、性别、民族等字段。 |            |        |          |             |                                                                          |
| 字段名             | 字段名注释                                                      | 是否字典列 | 有数据 | 数据来源 | 数据类型    | 数据抓取后处理方式                                                       |
| Patient_ID         | 患者ID，患者信息表主键                                          |            | Y      |          | varchar(38) | 抓取原系统的Patient_ID字段，后续该列会根据Patient_ID列处理方式进行扩展。 |
| Sex                | 患者性别                                                        |            |        |          | varchar(30) | 抓取原始系统的数据。                                                     |
| Birth_Date         | 患者生日                                                        |            |        |          | datetime    | 抓取原始系统的数据。不脱敏。                                             |
| Marital_Status     | 婚姻状况                                                        |            |        |          | varchar(30) | 对婚姻字段做标准化。                                                     |
| Race               | 种族/民族                                                       |            |        |          | varchar(30) | 对民族字段做标准化。                                                     |
| RAW_Race           | 种族/民族原始值                                                 | Y          |        |          | varchar(30) | 抓取原系统的数据。                                                       |
| RAW_Sex            | 性别原始值                                                      | Y          |        |          | varchar(30) | 抓取原系统的数据。                                                       |
| RAW_Birth_Date     | 患者生日原始值,只处理原始数据库日期为字符类型的数据             |            |        |          | varchar(30) |                                                                          |
| RAW_Marital_Status | 婚姻状况原始值                                                  | Y          |        |          | varchar(30) | 抓取原系统数据。                                                         |
| Patient_Name       | 患者姓名                                                        |            |        |          | varchar(50) | 抓取原系统的数据。                                                       |
| Patient_Phone      | 手机或电话                                                      |            |        |          | varchar(30) | 抓取原系统的数据。                                                       |
| Patient_ID_Card    | 患者身份证                                                      |            |        |          | varchar(30) | 对原数据不合法的，进行置空。                                             |
| Provider_ID        | 医疗机构编码                                                    |            | Y      |          | varchar(30) | 医院编号。                                                               |
| Update_Datetime    | 记录抓取到PDCDM的时间戳，用于后续增量处理数据                   |            | Y      |          | datetime    | 数据抓取时间戳，用于标记该记录抓取时间。                                 |

##附：（济南二院例）
| jn_n2_his_comm.marital_status_dict |                    | 婚姻字典表 |    |      |
|------------------------------------|--------------------|------------|----|------|
|                                    |                    |            |    |      |
| patient                            | RAW_Marital_Status | 1          | WH | 未婚 |
| patient                            | RAW_Marital_Status | 2          | YH | 已婚 |
| patient                            | RAW_Marital_Status | 3          | SO | 丧偶 |
| patient                            | RAW_Marital_Status | 4          | LH | 离婚 |
| patient                            | RAW_Marital_Status | 9          | QT | 其他 |
##设计要点:						
<pre>						
描述该表的一些辅助字段。


</pre>						

##本表数据处理需求点：												
<pre>	
例如二院：				
1.对Patient_ID列进行扩展处理。
2.对性别、婚姻和民族做字典化处理。民族字段中很多用了单字的简写方式。保留相关解析时的字典信息。
3.身份证号存在不合法需置空。

						
</pre>				
##本表数据存在的问题：						
<pre>						
描述具体表存在的一些问题。
例如二院：
1.2013.9.1日后入院的门诊患者中，至少有21835例患者身份证信息存在错误（位数不是15或者18位），  
2017.1.1后有13141例，例如patient_id为0000061936,  00032109,0000069630等，  
其中主要有身份证尾号少了一位和、填写了一位数字或填了无意义的信息，填了患者生日等几种错误。  
2013.9.1后入院的住院患者有1332例患者身份证信息存在错误（位数不是15或者18位），  
2017.1.1后有163例，例如patient_id为0000006382,0000002280,等，  
中主要有身份证尾号少了一位、填写了一位数字或填无等几种错误。  
【参考jn_n2_his_medrec.pat_master_index表，jn_n2_his_outpadm.clinic_master表，jn_n2_his_medrec.pat_visit表】。	


</pre>					
						
##数据处理要点：						
<pre>						
描述该表需处理的字段以及写清如何处理。（最终处理结果）
例如二院：
全局数据处理要点：
1. 对于日期类字段，如果原始值为date或者datetime格式，直接填入对应字段内；如果原始值为字符串，则填入新添加的RAW字段内。  
这样处理可以简化对日期字段的处理。
2. Provider_ID主要是指医院的代码，在对医院分析前进行指定。需要保证不同医院拥有不同的代码。当前医院代码为'10002'。
3. Patient_ID和Visit_ID前需要添加医院代码+'_' ,用于保证多家医院数据融合后数据的全局唯一性。若一家医院有多个并行的系统或者分院，  
4. 该2个字段存在重复现象，需要对不同的系统的医院代码添加额外的后缀，如'10002_0_' 。Visit_ID要扩展为'10002_0_${Patient_ID}_${Visit_ID}'
4. 对于字符串类型的数据，需要做相应的预处理。包括对字符前后的空白字符做截取操作，对文本中的两类换行进行替换，便于导出csv时数据的处理。
5.注意RAW字段数据类型为字符串，非RAW字段为数值或时间列时，对数据类型要做类型判断和异常处理。
本表数据处理要点：
1.对身份证号长度做校验，剔除其中非法的身份证号；对民族字段为单个汉字的进行处理。	

					
</pre>
						
##数据增量抽取后DDL语句：(可根据实际情况进行修改)		
				
<pre>
CREATE TABLE IF NOT EXISTS `patient` (
  `Patient_ID` varchar(38) NOT NULL COMMENT 'Patient 表中的患者唯一标识，患者个人ID',
  `Sex` varchar(8) DEFAULT NULL COMMENT '患者性别',
  `Birth_Date` datetime DEFAULT NULL COMMENT '患者生日,不包括日期,脱敏后只包括年份和月份',
  `Marital_Status` varchar(8) DEFAULT NULL COMMENT '婚姻状况,已婚,未婚',
  `Race` varchar(8) DEFAULT NULL COMMENT '种族/民族',
  `Raw_Race` varchar(20) DEFAULT NULL COMMENT '种族/民族',
  `Raw_Sex` varchar(20) DEFAULT NULL COMMENT '性别原始值',
  `RAW_Birth_Date` varchar(30) DEFAULT NULL COMMENT '患者生日原始值，只处理原始数据库日期为字符类型的数据',
  `RAW_Marital_Status` varchar(30) DEFAULT NULL COMMENT '患者婚姻状况原始值',
  `Patient_Name` varchar(38) DEFAULT NULL COMMENT '患者姓名原始值',
  `Patient_Phone` varchar(38) DEFAULT NULL COMMENT '患者手机或电话原始值',
  `Patient_ID_Card` varchar(38) DEFAULT NULL COMMENT '患者身份证号原始值',
  `Provider_ID` varchar(30) DEFAULT NULL COMMENT '医疗机构编码',
  `Update_Datetime` datetime DEFAULT NULL COMMENT '记录更新时间',
  PRIMARY KEY (`Patient_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='患者信息表';				
</pre>				
##最终CDM的DDL语句(PDCDM扩展模型)：				
				
<pre>
CREATE TABLE IF NOT EXISTS `patient` (
  `Patient_ID` varchar(50) NOT NULL COMMENT 'Patient 表中的患者唯一标识，患者个人ID',
  `Sex` varchar(8) DEFAULT NULL COMMENT '患者性别',
  `Birth_Date` datetime DEFAULT NULL COMMENT '患者生日,不包括日期,脱敏后只包括年份和月份',
  `Marital_Status` varchar(8) DEFAULT NULL COMMENT '婚姻状况,已婚,未婚',
  `Race` varchar(8) DEFAULT NULL COMMENT '种族/民族',
  `Raw_Race` varchar(20) DEFAULT NULL COMMENT '种族/民族',
  `Raw_Sex` varchar(20) DEFAULT NULL COMMENT '性别原始值',
  `RAW_Marital_Status` varchar(30) DEFAULT NULL COMMENT '患者婚姻状况原始值',
  `Birth_Date_MD5` varchar(38) DEFAULT NULL COMMENT '患者生日MD5值',
  `Name_MD5` varchar(38) DEFAULT NULL COMMENT '患者姓名MD5值',
  `Phone_MD5` varchar(38) DEFAULT NULL COMMENT '患者手机或电话MD5值',
  `ID_Card_MD5` varchar(38) DEFAULT NULL COMMENT '患者身份证号MD5值',
  `Provider_ID` varchar(30) DEFAULT NULL COMMENT '医疗机构编码',
  `Update_Datetime` datetime DEFAULT NULL COMMENT '记录更新时间',
  PRIMARY KEY (`Patient_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='患者信息表';"				
</pre>			
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				

				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
				
						
						
						
		
						
						
						
						
						
		
						
						
						
						
						
						
						
						
						

