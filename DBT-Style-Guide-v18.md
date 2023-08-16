
# DBT Style Guide for DPP

This document can serve as a guide for naming conventions and style when using DBT for DPP projects. Applying this style guide to your DBT projects provides various benefits, including the following -

\- Applying best practices on naming and style for your DBT projects based on inputs from git-labs, dbt-labs and flipside

\- Usage of nomenclature associated with the Medallion architectural construct and various DPP specific naming standards 

\- Promoting consistency for automation of your DBT projects

\- Conventions for various DBT artifacts including models,macros,CTE's,references and various database objects used in the Snowflake environment.






## 1: Database Naming Conventions

### Database naming convention: <env>_<domain>

 Examples of database naming conventions:

        - LATEST_HOTELS_AND_LODGING
		- STAGE_HOTELS_AND_LODGING
		- LOAD_HOTELS_AND_LODGING
        - PROD_HOTELS_AND_LODGING

Note: No <layer> will be used in the naming convention for databases as this will help with blue-green deployments.


## 2: Schema Naming Conventions

Silver will reflect  source system as part of the name. Gold will reflect domain and hence will be source system agnostic. 

Note: Each Medallion layer will be represented using a three-character convention:

- BRZ for Bronze

- SIL for Silver layer

- GLD for Gold Layer

### Silver Schema Naming Convention:

  	
    <layer=SIL>_<source_system>_<subject-area>_<suffix>--optional

#### Silver schema naming examples:
       					
    - SIL_DREAMS_ROOM_RESERVATION
    - SIL_DREAMS_ROOM_RESERVATION_WORKING
    - SIL_TBX_ROOM_RESERVATION
    - SIL_TBX_ROOM_RESERVATION_WORKING

    Note: <suffix> will provide an optional description of the underlying process such as '_WORKING'.

### Gold Schema Naming Convention:
    
    <layer=GLD>_<subject-area>_<suffix>--optional

#### Gold schema naming examples:

     - GLD_ROOM_RESERVATION
     - GLD_ROOM_RESERVATION_WORKING

    Note: <suffix> will provide an optional description of the underlying process such as '_WORKING'. This can be modified as needed to match a suitable description for the GOLD layer.







## 3: Table Conventions


### Table naming convention: 

Use <prefix>-<entity>-<suffix>_optional

    -   <prefix>  describes type of table:

			- FCT_  for fact tables
			- DIM_ for dimension tables
            - INT_ for intermediate tables

    -   <entity>  describes an entity name and provides a business description for the underlying process

		- Examples
			 - RESERVATION
			 - TC_GRP
    -   <suffix> This is optional and provides a description of the output of the underlying work action involved in producing the table.

			- Examples
				. _CLEANSED
				. _VERSIONED
				. _MONTHLY
                . _QUARTERLY          


-   Examples

		- DIM_DESTINATION
		- FCT_RESERVATION
		- INT_RESERVATION_VERSIONED
        - INT_RESERVATION_CLEANSED
## 4: Model Conventions

All model names should be in snake case.

### DBT Model naming conventions: 

- Use the same convention used for tables -
<prefix>__<entity>_<suffix>_optional

        - Refer to the table section for additional information

- All models in silver should select from a reference : {{ ref }} function

     - example:

        select * from {{ ref('tc_gst_cleansed') }}


- Only models in bronze should select from sources
    - example:

        select * from 
        {{ source. ('brz_dreams_rooms_reservations', 'tc_grp') }}
    - This will help with the following -
        - Help define the lineage of your data
        - Test any assumptions about source data
        - Calculate the freshness of source data when needed
        

- Specify the default materilization behavior explicitly in the configuration section of the model at the top
    -  example: {{ config(materialized='table') }}

    - Models within the silver folder should be materialized as table or incremental. Check the design/source-to-stage document for specifics. 

-  Use a two-argument ref any time you are referencing a model defined in a different package or project. 
select * from {{ ref('project_or_package', 'model_name') }}


### Model folder conventions: 

	• For all  Silver models - 
		 - create models under /models/silver
	• For all Gold models - 
        -  create models under /models/gold



### Model Configs
- Model configurations at the folder level should be considered (and if applicable, applied) first.
- More specific configurations should be applied at the model level using one of these methods.    



### Grouping attributes inside Models

Where possible, 
columns should be ordered in categories, where identifiers are first and date/time fields are at the end.
Example:

(

        select
--ids 

	    ,order_id
        ,customer_id

--dimensions

	    ,order_status
        ,is_shipped

--measures

		,order_total

--date/times

		,created_at
        , updated_at

--metadata_

		, _ sdc_batched_at
    from source
)


### More Model naming considerations

- Limit use of abbreviations that are related to domain knowledge. An onboarding employee will understand current_order_status better than current_os.

- Use names based on the business terminology, rather than the source terminology.

- Each model should have a primary key that can identify the unique row, and should be named <object>_id, e.g. account_id – this makes it easier to know what id is being referenced in downstream joined models.

- If a surrogate key is created, it should be named <object>_sk.

- Consistency is the key! Use the same field names across models where possible.

    - Example: a key to the customers table should be named customer_id rather than user_id.

## 5: CTE Conventions

Usage of CTE's (Common Table Expressions aka WITH statement) is recommended in DBT as they make SQL more readable and performant. 

- CTE example

    	WITH important_list_cte AS 
		(SELECT DISTINCT specific_column FROM other_table
		WHERE specific_column!='foo') 
		SELECT primary_table.column_1,primary_table.column_2 
        FROM primary_table 
        INNER JOIN 
        important_list_cte
        ON primary_table.column_3 = important_list_cte.specific_column


### Naming Convention for CTE expressions-

    - Use suffix _cte  for all CTE expressions for readability
    - It is recommended to apply snake case to the name of CTE expression 

### General Guidelines  for CTE expressions-
- CTEs should be placed at the top of the query

- Use CTEs to reference other tables

- CTE's fall into two broad categories as described below. It is       
  recommended that the type of CTE is captured as part of the documentation.

    - Import: Used to bring data into a model. These are kept relatively  
      simple and refrain from complex operations such as joins and column 
      transformations

    - Logical: Used to perform a logical step with the data that is brought 
      into the model toward the end result

- Where performance permits, CTEs should perform a single, logical unit of  
  work

- CTE names should be as concise as possible while still being clear

- Avoid long names like 'replace_sfdc_account_id_with_master_record_id'  and prefer a shorter name with a comment in the CTE. This will help avoid table aliasing in joins.

- CTEs with confusing or notable logic should be commented inline in the  file and documented in dbt docs

- CTEs that are duplicated across models should be pulled out into their own models

-  All {{ ref() }} or {{ source() }} statements should be placed within import CTEs so that dependent model references are easily seen and located

- Where applicable, opt for filtering within import CTEs over filtering within logical CTEs. This allows a developer to easily see which data contributes to the end result.


- SQL should end with a simple select statement. All other logic should be contained within CTEs to make stepping through logic easier while troubleshooting. Example: select * from final


### Structure of CTE expressions

- SQL and CTEs within a model should follow this structure:

		. With statement
		. Import CTEs
		. Logical CTEs
		. Simple select statement
        
	• Example CTE
		 -- Jaffle shop went international!
		with
		
		-- Import CTEs
		regions as (
		    select * from {{ ref('stg_jaffle_shop__regions') }}
		),
		
		nations as (
		    select * from {{ ref('stg_jaffle_shop__nations') }}
		),
		
		suppliers as (
		    select * from {{ ref('stg_jaffle_shop__suppliers') }}
		),
		
		-- Logical CTEs
		locations as (
		    select
		        {{ dbt_utils.generate_surrogate_key([
		            'regions.region_id',            
		            'nations.nation_id'
		        ]) }} as location_sk,
		        regions.region_id,
		        regions.region,
		        regions.region_comment,
		        nations.nation_id,
		        nations.nation,
		        nations.nation_comment
		    from regions
		    left join nations
		        on regions.region_id = nations.region_id
		),
		
		final as (
		    select
		        suppliers.supplier_id,
		        suppliers.location_id,
		        locations.region_id,
		        locations.nation_id,
		        suppliers.supplier_name,
		        suppliers.supplier_address,
		        suppliers.phone_number,
		        locations.region,
		        locations.region_comment,
		        locations.nation,
		        locations.nation_comment,
		        suppliers.account_balance
		    from suppliers
		    inner join locations
		        on suppliers.location_id = locations.location_sk
		)
		
		-- Simple select statement
		select * from final




## 6:Datatype Conventions

This section refers to conventions to be used when applying datatypes to attributes in DBT.

- Datatype recommendations for DBT

		- Use NUMBER instead of DECIMAL, NUMERIC, INTEGER, BIGINT, etc.
		- Use FLOAT instead of DOUBLE, REAL, etc.
		- Use VARCHAR instead of STRING, TEXT, etc.
		- Use TIMESTAMP instead of DATETIME
			 - Would recommend using TIMESTAMP_LTZ as is provides a local 
               time zone in UTC format
            -  Note: The default for TIMESTAMP is TIMESTAMP_NTZ which does 
               not include a time zone.

-  Apply default data types and not aliases 





## 7: Reference/Join Conventions

This section provides recommendations on reference/join conventions in DBT.

- Reference the full table name instead of an alias when the table name is shorter, maybe less than 20 characters. 
         -  Note: Try to rename the CTE if possible, and lastly consider 
                aliasing to something descriptive)
- Always qualify each column in the SELECT statement with the table name / alias for easy navigation

- Only use double quotes when necessary, such as columns that contain special characters or are case sensitive.
	      - Example: SELECT "First_Name_&_" AS first_name,
	
- Recommend accessing JSON using the bracket syntax.

	      - Example:

            - SELECT data_by_row['id']::bigint as id_value...
- Recommend using explicit join statements

	      - Example:
            - SELECT* FROM first_table INNER JOINs second_table

- Do not apply the USING command in joins because it produces inaccurate results in Snowflake


### Reference/Join example:

		SELECT

		,budget_forecast.account_id,date_details.fiscal_year,date_details

        ,fiscal_quarter,date_details.fiscal_quarter_name,cost_category

        ,cost_category_level_1,cost_category.cost_category_level_2

		FROM 

		budget_forecast_cogs_opex 

        AS

		budget_forecast

		LEFT JOIN date_details 

        ON 

        date_details.first_day_of_month=budget_forecast.accounting_period 

        LEFT JOIN 

        cost_category 

        ON 

        budget_forecast.unique_account_name=cost_category.unique_account_name
		

## 8: Column Conventions

This section provides conventions to apply for columns in DBT.

### ID fields


- End with suffix _id or _pk or _sk
		    •  Example:
		    SELECT id AS account_id,


### Metric/measure fields            
		

- Price/revenue fields should be in decimal currency 
	- (e.g. 19.99 for $19.99; many app databases store prices as 
              integers in cents). 
	- If non-decimal currency is used, indicate this with suffix. e.
              g. price_in_cents.

- More info on metrics/kpi's
	- Metrics names must begin with a letter, cannot contain 
                  whitespace, and should be all lowercase.
	- Tags and/or Meta properties should match the categories above and be 
     used to organize metrics at the category or business function level.
	- Meta properties should be used to track metric definition ownership.
		- Example metric

					- name: promoters_pct
                      label: Percent Promoters (Cloud)
                      description: 'The percent of dbt Cloud users in the    
                      promoters segment.'
                      tags: ['Company Metric']
                      calculation_method: derivedexpression: 
                      "{{metric('base__count_nps_promoters_cloud')}} 
                      / {{metric('base__total_nps_respondents_cloud')}}     
                      "timestamp: created_attime_grains: [day, month,       
                      quarter, year]meta:
                      metric_level: 'Company'owner(s): 'Jane Doe'



### Timestamp fields

-   Timestamp fields should end with _at and should always be in UTC.
		• Recommend using TIMESTAMP_LTZ as the datatype



### Date Fields

- Dates should end with _date.

- Date/time columns should be named according to these conventions:
			- Timestamps: <event>_at
              Format: UTC
              Example: created_at

			- Dates: <event>_date
              Format: Date
             Example: created_date
		
- Avoid key words like 'date' or 'month' as a column name.

- When truncating dates, name the column in accordance with the truncation.
			
        SELECT
		original_at,-- 2020-01-15 12:15:00.00
        original_date, -- 2020-01-15
        DATE_TRUNC('month',original_date)
        AS
        original_month -- 2020-01-01
			
			
			
### Boolean Fields

- Booleans should be prefixed with is_ or has_.

    - Example: 
        - is_active_customer and has_admin_access




### Ambiguous Fields

- An ambiguous field name such as id, name, or type should always be prefixed by what it is identifying or naming:
		- Example:

		SELECT id AS account_id,
		Name AS account_name,
		Type AS account_type,


### Case for Naming columns

- All field names should be snake-cased:

		  - Example:

		-       SELECT dvcecreatedtstamp As device_created_timestamp
		
	
 ### Column Aliasing

- Use the AS operator when aliasing a column or table.
    - Example:
        - SELECT id AS account_id,

## 9:Function Conventions

This section provides recommendations on conventions when using functions in DBT.

- For better readability and simplicity, recommend using IFNULL to NVL function in DBT

- Recommend usage of an IFF statement over a CASE statement:

		- Example:
			SELECT
			IFF(column_1='foo',column_2,column_3)
			AS
            logic_switch,…

            The above IFF statement is preferred over the CASE statement 
            equivalent below:

            SELECT
            CASE
            WHEN column_1 = 'foo' THEN column_2
            ELSE column_3
            END AS logic_switch,
## 10: YAML and Markdown Conventions

This section provides conventions to apply when using YAML and markdowns (.md) in DBT.

- Every subdirectory contains their own .yml file(s) which contain configurations for the models within the subdirectory.

- YAML and markdown files should be prefixed with an underscore ( _ ) to keep it at the top of the subdirectory.


### YAML Naming Conventions

- YAML and markdown files should be named with the following conventions 
    _<description>__<config>

        - <Description> is typically the folder of models you're setting 
      configurations for.

            Examples: core, staging, intermediate

        - <config> is the top-level resource you are configuring.

            Examples: docs, models, sources


  -  _<name_of_the_source>__models.yml 
        -  Example:
            _ salesforce_customers__sources.yml 
  -  _<name_of_the_source>__sources.yml
        - Example:
            _salesforce_customers__docs.md




### Additional YAML Conventions

- Indents should use two spaces.
- List items should be indented.
- Use a new line to separate list items that are dictionaries, where appropriate.
- Lines of YAML should be no longer than 80 characters.
- Items listed in a single .yml or .md file should be sorted alphabetically for ease of finding in larger files.



- Each top-level configuration should use a separate .yml file (i.e, sources, models) Example:

models
├── marts
└── staging
    └── jaffle_shop
        ├── _jaffle_shop__docs.md

        ├── _jaffle_shop__models.yml
        ├── _jaffle_shop__sources.yml
        ├── stg_jaffle_shop__customers.sql
        ├── stg_jaffle_shop__orders.sql
        └── stg_jaffle_shop__payments.sql
dbt_project.yml configurations should be prefixed with + to avoid namespace collision with directories.

Example:

models:
  my_project:
    marts:
      +materialized: table



### Example YAML

_jaffle_shop__models.yml:

version: 2

models:

  - name: base_jaffle_shop__nations

    description: This model cleans the raw nations data
    columns:
      - name: nation_id
        tests:
          - unique
          - not_null   

  - name: base_jaffle_shop__regions
    description: >
      This model cleans the raw regions data before being joined with nations
      data to create one cleaned locations table for use in marts.
    columns:
      - name: region_id
        tests:
          - unique
          - not_null

  - name: stg_jaffle_shop__locations

    description: "{{ doc('jaffle_shop_location_details') }}"

    columns:
      - name: location_sk
        tests:
          - unique
          - not_null



### Example Markdown

_jaffle_shop__docs.md:

  {% docs enumerated_statuses %}
    
    Although most of our data sets have statuses attached, you may find some
    that are enumerated. The following table can help you identify these statuses.
    | Status | Description                                                                 |
    |--------|---------------|
    | 1      | ordered       |
    | 2      | shipped       |
    | 3      | pending       |
    | 4      | order_pending | 

    
{% enddocs %}

{% docs statuses %} 

    Statuses can be found in many of our raw data sets. The following lists
    statuses and their descriptions:
    | Status        | Description                                                                 |
    |---------------|-----------------------------------------------------------------------------|
    | ordered       | A customer has paid at checkout.                                            |
    | shipped       | An order has a tracking number attached.                                    |
    | pending       | An order has been paid, but doesn't have a tracking number.                 |
    | order_pending | A customer has not yet paid at checkout, but has items in their cart. | 

{% enddocs %}

## 11: Jinja Conventions

This section provides conventions to apply when using Jinja in DBT.


- Jinja delimiters should have spaces inside of the delimiter between the brackets and your code.

    - Example: 

        {{ this }} instead of {{this}}



-   Use whitespace control to make compiled SQL more readable.

-   An effort should be made for a good balance in readability for both 
    templated and compiled code. However, opt for code readability over 
    compiled SQL readability when needed.

-   A macro file should be named after the main macro it contains.

-   A file with more than one macro should follow these conventions:

        - There is one macro which is the main focal point:

        -  The file is named for the main macro or idea

        -  All other macros within the file are only used for the purposes of 
           the main idea and not used by other macros outside of the file.

-   Use new lines to visually indicate logical blocks of Jinja or to enhance readability.



###  Jinja Example:


{%- set orig_cols = adapter.get_columns_in_relation(ref('fct_orders')) %}

{%- set new_cols = dbt_utils.star(
      from=ref('fct_order_items'),
      except=orig_cols
) %}

-- original columns. 

{{ col }} is indented here, but choose what will satisfy

-- your own balance for Jinja vs. SQL readability. 

{%- for col in orig_cols %}
      {{ col }}
{% endfor %}

-- column difference

{{ new_cols }}


- Use new lines within Jinja delimiters and arrays if there are multiple arguments.

    - Example:

{%- dbt_utils.star(

      from=ref('stg_jaffle_shop__orders'),
      
      except=[
          'order_id',
          'ordered_at',
          'status'
      ],
      prefix='order_'
) %}





## 12: SQL Conventions


This section provides conventions to apply when using SQL statements in DBT.


- Use leading  commas instead of training commas when specifying multiple columns

- DO NOT OPTIMIZE FOR FEWER LINES OF CODE. New lines are cheap, brain time is 
  expensive; new lines should be used within reason to produce code that is 
  easily read.


-   Indents should use four spaces.

- When dealing with long 'when' or 'where' clauses, predicates should be on a new line and indented.

    Example:

    where user_id is not null
    and status ='pending' 
    and location ='hq'


- Lines of SQL should be no longer than 80 characters and new lines should be used to ensure this.

    Example:
    sum(
    case
        when order_status ='complete'then order_total 
    end
    ) asmonthly_total,

    {{ get_windowed_values(
      strategy='sum',
      partition='order_id',
      order_by='created_at',
      column_list=[
          'final_cost']
    ) }} astotal_final_cost


-  Use all lowercase unless a specific scenario needs you to do otherwise. This means that keywords, field names, function names, and file names should all be lowercased.


- The as keyword should be used when aliasing a field or table

- Fields should be stated before aggregates / window functions


- Aggregations should be executed as early as possible before joining to another table.

- Ordering and grouping by a number (eg. group by 1, 2) is preferred over listing the column names (see this rant for why). 

-  Note that if you are grouping by more than a few columns, it may be worth revisiting your model design. If you really need to, the dbt_utils.group_by function may come in handy.


-   Prefer union all to union *

-   Avoid table aliases in join conditions (especially initialisms) – it's harder to understand what the table called "c" is compared to "customers".

- If joining two or more tables, always prefix your column names with the table alias. If only selecting from one table, prefixes are not needed.

- Be explicit about your join (i.e. write inner join instead of join). left joins are the most common, right joins often indicate that you should change which table you select from and which one you join to.

- Avoid the using clause in joins, preferring instead to explicitly list the CTEs and associated join keys with an on clause.

-   Joins should list the left table first (i.e., the table you're joining data to)

    Example:

    select 

    trips.*,

    drivers.rating as driver_rating,

    riders.rating as rider_rating

    from trips

    left join users as drivers

    on trips.driver_id=drivers.user_id

    left join  users as riders

    on trips.rider_id=riders.user_id


### SQL Example

	With
	my_data 
    as
    (
    select *
    from{{ ref('my_data') }}
    where not is_deleted
    ),
	some_cte as
    (
    select *
    from{{ ref('some_cte') }}
    ),
	some_cte_agg as(
    select
    id,
    sum(field_4) astotal_field_4,
    max(field_5) asmax_field_5
    from
    some_cte
    group by 1),

	final 
    as(
    select [distinct]
    my_data.field_1,
    my_data.field_2,
    my_data.field_3,

	--use line breaks to visually separate calculations into blockscase

    When my_data.cancellation_date is null
    and 
    my_data.expiration_date is not null
    then expiration_data
    When my_data.cancellation_date is null 
    then my_data.start_date+7
    else
    my_data.cancellation_dateend as cancellation_date,
	some_cte_agg.total_field_4,
    some_cte_agg.max_field_5
    from my_data
    left join some_cte_agg  
    on my_data.id=some_cte_agg.id
    where my_data.field_1='abc' 
    and (
    my_data.field_2='def'ormy_data.field_2='ghi')
    qualify 
    row_number() 
    over (
    partition by my_data.field_1
    order by my_data.start_date
    desc) =1)
    
    select * from final

    



## 13: View Conventions

This section provides conventions to apply when using views in DBT.


- Use naming convention view_<suffix> to name view objects in DBT
    -   <suffix> can be a meanigful business description of underlying table 
        name/s 

- Use explicit 'materialized = view' in config statement of the underlying  
  model
   -  Example:

        {{ config(materialized='view') }}



- Use Convenience Views (ez_) as needed to combine facts and dimensions as needed. 
    -   The convenience view with ez_<suffix>
        - The <suffix> could be a meaningful business description of the 
          underlying fact and dimension tables represented.
    -   This should only be views in the gold layer.
    -   This could be done for reasons such as when curation is doing more of 
        the “lift” for the view.

## 14: Comments Conventions

This section provides conventions to apply when using comments within DBT.

-   When making single line comments in a model use the -- syntax
-   When making multi-line comments in a model use the /* */ syntax
## 15: Documentation Considerations

This section provides conventions to apply when using documentation in DBT.

--Coming Soon
## 16: Tagging Considerations

This section provides conventions to apply when using tagging  in DBT.

--Coming Soon
## 17: General Best Practices

This section provides general best practices for conventions in DBT.

-   No tabs should be used - only spaces. Your editor should be setup to convert tabs to spaces - see our onboarding template for more details.

-   Wrap long lines of code, between 80 and 100, to a new line.

-   Do not use the USING command in joins because it produces inaccurate results in Snowflake. Create an account to view the forum discussion on this topic.

-   Understand the difference between the following related statements and use appropriately:

		- UNION ALL and UNION
		- LIKE and ILIKE
		- NOT and ! and <>
		- DATE_PART() and DATE_TRUNC()

- Use the AS operator when aliasing a column or table.

- Prefer DATEDIFF to inline additions date_column + interval_column. The function is more explicit and will work for a wider variety of date parts.

-   Prefer != to <>. This is because != is more common in other programming languages and reads like "not equal" which is how we're more likely to speak.


- Prefer LOWER(column) LIKE '%match%' to column LIKE '%Match%'. 
    -   This lowers the chance of stray capital letters leading to an unexpected result.

- Prefer WHERE to HAVING when either would suffice.
## Credits and References

This section lists the reference material used to create this dbt style guide.


List of Reference Material used:-


https://about.gitlab.com/handbook/business-technology/data-team/platform/sql-style-guide/#naming-conventions

https://github.com/dbt-labs/corp/blob/main/dbt_style_guide.md#model-file-naming-and-coding

https://docs.flipsidecrypto.com/contribute/model-standards

https://github.com/dbt-labs/corp/blob/main/dbt_style_guide.md
 
https://about.gitlab.com/handbook/business-technology/data-team/platform/dbt-guide/#style-and-usage-guide
 
https://docs.flipsidecrypto.com/contribute/model-standards
 
https://gist.github.com/fredbenenson/7bb92718e19138c20591
 
https://github.com/mattm/sql-style-guide

https://docs.flipsidecrypto.com/contribute/model-standards