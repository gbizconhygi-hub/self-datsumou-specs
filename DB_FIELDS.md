# DB_FIELDS.md (auto-generated)
_source: information_schema • db: xs608427_selfdatsumoucore • generated: 2025-11-13 12:44:45_

## ai_answer_templates
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|category_code|varchar(64)|YES|MUL|NULL|||
|faq_id|int(11)|YES||NULL|||
|channel|enum('web','line','any')|YES||'any'|||
|template_key|varchar(64)|NO||NULL|||
|title|varchar(255)|YES||NULL|||
|body_text|longtext|NO||NULL|||
|variables_json|longtext|YES||NULL|||
|priority|int(11)|YES||100|||
|is_active|tinyint(1)|YES||1|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## ai_categories
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|code|varchar(64)|NO|UNI|NULL|||
|name|varchar(128)|NO||NULL|||
|description|text|YES||NULL|||
|priority|int(11)|YES||100|||
|is_active|tinyint(1)|YES||1|||
|created_at|datetime|YES||current_timestamp()|||

## ai_category_terms
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|category_code|varchar(64)|NO|MUL|NULL|||
|term_type|enum('positive','negative','urgent','boost')|NO||NULL|||
|term|varchar(128)|NO||NULL|||
|weight|int(11)|YES||1|||
|created_at|datetime|YES||current_timestamp()|||

## ai_config_audits
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|actor|varchar(64)|YES||NULL|||
|action|varchar(64)|NO||NULL|||
|target|varchar(128)|NO|MUL|NULL|||
|before_value|longtext|YES||NULL|||
|after_value|longtext|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## ai_prompt_snippets
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|snippet_key|varchar(64)|NO|UNI|NULL|||
|content|longtext|NO||NULL|||
|version|int(11)|YES||1|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## ai_settings
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|setting_key|varchar(64)|NO|UNI|NULL|||
|setting_value|text|YES||NULL|||
|updated_by|varchar(64)|YES||NULL|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## ai_tone_policies
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|channel|enum('web','line')|NO|MUL|NULL|||
|key_name|varchar(64)|NO||NULL|||
|value_text|text|NO||NULL|||
|is_active|tinyint(1)|YES||1|||

## audit_logs
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|user_id|int(11)|YES||NULL|||
|role|enum('admin','owner','customer','system')|YES||NULL|||
|action|varchar(255)|NO||NULL|||
|entity|varchar(255)|YES||NULL|||
|entity_id|int(11)|YES||NULL|||
|details|longtext|YES||NULL|||
|ip|varchar(64)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## failed_logins
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|bigint(20)|NO|PRI|NULL|auto_increment||
|user_ref|varchar(128)|NO|MUL|NULL|||
|ip|varbinary(16)|YES||NULL|||
|ts|timestamp|NO||current_timestamp()|||

## fee_items
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|code|varchar(32)|YES|UNI|NULL|||
|name|varchar(255)|YES||NULL|||
|default_unit_price|int(11)|YES||0|||
|billing_cycle|enum('monthly','one_time')|YES||'monthly'|||
|created_at|datetime|YES||current_timestamp()|||

## fee_packages
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|code|varchar(32)|YES|UNI|NULL|||
|name|varchar(255)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## hq_chat_links
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|thread_id|int(11)|NO|MUL|NULL|||
|link_type|enum('ticket','request')|NO||NULL|||
|link_id|int(11)|NO||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## hq_chat_messages
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|thread_id|int(11)|NO|MUL|NULL|||
|sender_role|enum('owner','admin','ai')|NO||NULL|||
|message|text|YES||NULL|||
|attachments_json|longtext|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## hq_chat_threads
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|subject|varchar(255)|YES||NULL|||
|status|enum('open','assigned','closed')|YES||'open'|||
|last_message_at|datetime|YES||current_timestamp()|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## hq_meetings
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|slot_id|int(11)|NO|MUL|NULL|||
|shop_id|int(11)|NO|MUL|NULL|||
|requested_by|varchar(64)|NO||NULL|||
|status|enum('requested','confirmed','cancelled')|YES||'confirmed'|||
|memo|varchar(255)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## hq_meeting_slots
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|slot_start|datetime|NO|MUL|NULL|||
|slot_end|datetime|NO||NULL|||
|mode|enum('online','phone','visit')|YES||'online'|||
|capacity|int(11)|YES||1|||
|status|enum('available','closed')|YES|MUL|'available'|||
|notes|varchar(255)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## intents
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|template_code|varchar(64)|YES||NULL|||
|name|varchar(255)|YES||NULL|||
|faq_text|text|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## invoices
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|owner_id|int(11)|NO|MUL|NULL|||
|shop_id|int(11)|NO|MUL|NULL|||
|billing_month|char(7)|NO||NULL|||
|total_amount|int(11)|YES||0|||
|status|enum('draft','sent','paid','void')|YES||'draft'|||
|pdf_path|varchar(512)|YES||NULL|||
|sent_at|datetime|YES||NULL|||
|paid_at|datetime|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## invoice_items
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|invoice_id|int(11)|NO|MUL|NULL|||
|item_type|enum('royalty','system','option','supply','other')|YES||'other'|||
|ref_id|int(11)|YES||NULL|||
|description|varchar(255)|YES||NULL|||
|amount|int(11)|NO||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## jwt_revocations
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|jti|char(36)|NO|PRI|NULL|||
|exp_at|datetime|NO|MUL|NULL|||
|reason|varchar(64)|YES||NULL|||

## owners
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|owner_code|varchar(32)|YES|UNI|NULL|||
|owner_type|enum('corp','individual')|YES||'corp'|||
|owner_status|enum('active','suspended','closed')|YES||'active'|||
|contract_holder_name|varchar(255)|YES||NULL|||
|contract_type|varchar(64)|YES||NULL|||
|contract_zip|varchar(16)|YES||NULL|||
|contract_addr_line1|varchar(255)|YES||NULL|||
|contract_addr_line2|varchar(255)|YES||NULL|||
|contract_addr_building|varchar(255)|YES||NULL|||
|contract_phone|varchar(32)|YES||NULL|||
|corporate_number|varchar(32)|YES||NULL|||
|representative_name|varchar(255)|YES||NULL|||
|representative_kana|varchar(255)|YES||NULL|||
|representative_birthdate|date|YES||NULL|||
|representative_email|varchar(255)|YES||NULL|||
|mailing_zip|varchar(16)|YES||NULL|||
|mailing_addr_line1|varchar(255)|YES||NULL|||
|mailing_addr_line2|varchar(255)|YES||NULL|||
|mailing_addr_building|varchar(255)|YES||NULL|||
|mailing_attention|varchar(255)|YES||NULL|||
|mailing_phone|varchar(32)|YES||NULL|||
|chatwork_room_id|varchar(64)|YES||NULL|||
|preferred_contact|enum('portal','email','line')|YES||'portal'|||
|billing_email|varchar(255)|YES||NULL|||
|post0_contract|tinyint(1)|YES||0|||
|notes|text|YES||NULL|||
|is_profile_complete|tinyint(1)|YES||0|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## owner_contacts
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|owner_id|int(11)|NO|MUL|NULL|||
|contact_name|varchar(255)|YES||NULL|||
|contact_phone|varchar(32)|YES||NULL|||
|contact_email|varchar(255)|YES||NULL|||
|chatwork_id|varchar(64)|YES||NULL|||
|role_label|varchar(64)|YES||NULL|||
|is_active|tinyint(1)|YES||1|||
|created_at|datetime|YES||current_timestamp()|||

## package_components
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|package_id|int(11)|NO|MUL|NULL|||
|fee_item_id|int(11)|NO|MUL|NULL|||
|qty|int(11)|YES||1|||
|created_at|datetime|YES||current_timestamp()|||

## portal_users
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|role|enum('admin','owner')|NO||NULL|||
|shop_id|int(11)|YES||NULL|||
|login_id|varchar(64)|YES|UNI|NULL|||
|login_pass_hash|varchar(255)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## rate_limits
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|key_hash|char(64)|NO|PRI|NULL|||
|window_start|datetime|NO|PRI|NULL|||
|count|int(11)|NO||0|||

## request_templates
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|template_code|varchar(64)|YES|UNI|NULL|||
|name|varchar(255)|YES||NULL|||
|form_schema_json|longtext|YES||NULL|||
|auto_routes_json|longtext|YES||NULL|||
|billing_hook_json|longtext|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## sales_records
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|merchant_public_id|varchar(64)|YES|MUL|NULL|||
|shop_id|int(11)|YES|MUL|NULL|||
|source_type|enum('stores','manual','other')|YES||'stores'|||
|transaction_type|enum('sale','refund','trial')|YES||'sale'|||
|transaction_datetime|datetime|NO|MUL|NULL|||
|amount|int(11)|YES||0|||
|fee|int(11)|YES||0|||
|revenue|int(11)|YES||NULL|STORED GENERATED||
|product_name|varchar(255)|YES||NULL|||
|customer_name|varchar(255)|YES||NULL|||
|customer_email|varchar(255)|YES||NULL|||
|customer_phone|varchar(32)|YES||NULL|||
|is_trial|tinyint(1)|YES||0|||
|payment_method|varchar(64)|YES||NULL|||
|status|varchar(32)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## shops
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|code|varchar(32)|YES|UNI|NULL|||
|owner_id|int(11)|NO|MUL|NULL|||
|merchant_public_id|varchar(64)|YES||NULL|||
|name|varchar(255)|YES||NULL|||
|region_name|varchar(64)|YES||NULL|||
|contract_date|date|YES||NULL|||
|open_date|date|YES||NULL|||
|shop_status|enum('preopen','open','suspended','closed')|YES||'open'|||
|address_zip|varchar(16)|YES||NULL|||
|address_line1|varchar(255)|YES||NULL|||
|address_line2|varchar(255)|YES||NULL|||
|address_building|varchar(255)|YES||NULL|||
|phone|varchar(32)|YES||NULL|||
|shop_email|varchar(255)|YES||NULL|||
|remotelock_child_email|varchar(255)|YES||NULL|||
|business_hours|text|YES||NULL|||
|stores_public_url|varchar(512)|YES||NULL|||
|notes|text|YES||NULL|||
|billing_notes|text|YES||NULL|||
|royalty_type|enum('flat','percent')|YES||'flat'|||
|royalty_flat|int(11)|YES||0|||
|royalty_percent|decimal(5,2)|YES||0.00|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## shop_fee_addons
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|item_code|varchar(32)|YES||NULL|||
|unit_price|int(11)|YES||0|||
|qty|int(11)|YES||1|||
|billing_cycle|enum('monthly','one_time')|YES||'monthly'|||
|visible_on_invoice|tinyint(1)|YES||1|||
|start_date|date|YES||NULL|||
|end_date|date|YES||NULL|||
|notes|text|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## shop_fee_package_assignments
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|package_id|int(11)|NO|MUL|NULL|||
|start_date|date|YES||NULL|||
|end_date|date|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## shop_options
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|opening_hours|varchar(255)|YES||NULL|||
|holidays|varchar(255)|YES||NULL|||
|notes|text|YES||NULL|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## shop_requests
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|template_code|varchar(64)|YES||NULL|||
|status|enum('draft','submitted','approved','rejected')|YES||'draft'|||
|payload_json|longtext|YES||NULL|||
|attachments_json|longtext|YES||NULL|||
|requested_by|varchar(64)|YES||NULL|||
|ticket_id|int(11)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## shop_royalty_history
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|royalty_type|enum('flat','percent')|YES||'flat'|||
|royalty_flat|int(11)|YES||0|||
|royalty_percent|decimal(5,2)|YES||0.00|||
|start_date|date|YES||NULL|||
|end_date|date|YES||NULL|||
|notes|text|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## shop_secrets
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|UNI|NULL|||
|stores_api_key|varchar(255)|YES||NULL|||
|stores_api_secret|varchar(255)|YES||NULL|||
|line_channel_secret|varchar(255)|YES||NULL|||
|line_channel_access_token|varchar(255)|YES||NULL|||
|notes|text|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## shop_supplies
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|NO|MUL|NULL|||
|item_name|varchar(255)|YES||NULL|||
|unit_price|int(11)|YES||0|||
|qty|int(11)|YES||1|||
|status|enum('ordered','approved','delivered','billed')|YES||'ordered'|||
|created_at|datetime|YES||current_timestamp()|||

## tickets
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|YES|MUL|NULL|||
|source|enum('stores_form','customer_line','hp_form','portal_owner','portal_admin')|NO||NULL|||
|category|varchar(64)|YES||NULL|||
|subject|varchar(255)|YES||NULL|||
|status|enum('open','pending_owner','pending_admin','resolved','closed')|YES||'open'|||
|assigned_to|int(11)|YES||NULL|||
|customer_email|varchar(255)|YES||NULL|||
|customer_line_userid|varchar(64)|YES||NULL|||
|customer_phone|varchar(32)|YES||NULL|||
|last_sender|enum('customer','owner','admin')|YES||'customer'|||
|last_message_at|datetime|YES||current_timestamp()|||
|created_at|datetime|YES||current_timestamp()|||
|updated_at|datetime|YES||current_timestamp()|on update current_timestamp()||

## ticket_messages
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|ticket_id|int(11)|NO|MUL|NULL|||
|sender_role|enum('customer','owner','admin','system')|NO||NULL|||
|message|text|YES||NULL|||
|attachments_json|longtext|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## tutorial_feedbacks
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|log_id|int(11)|NO|MUL|NULL|||
|rating|enum('good','bad')|NO||NULL|||
|comment|varchar(255)|YES||NULL|||
|created_at|datetime|YES||current_timestamp()|||

## tutorial_logs
| Column | Type | Null | Key | Default | Extra | Comment |
|---|---|---|---|---|---|---|
|id|int(11)|NO|PRI|NULL|auto_increment||
|shop_id|int(11)|YES||NULL|||
|channel|enum('line','stores','hp')|YES|MUL|'line'|||
|user_identifier|varchar(128)|YES||NULL|||
|user_question|text|YES||NULL|||
|ai_answer|text|YES||NULL|||
|intent_label|varchar(64)|YES||NULL|||
|resolved|tinyint(1)|YES||0|||
|created_at|datetime|YES||current_timestamp()|||

## INDEXES

### ai_answer_templates
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|uq_cat_channel_key|0|1|category_code|BTREE|
|uq_cat_channel_key|0|2|channel|BTREE|
|uq_cat_channel_key|0|3|template_key|BTREE|
|idx_cat_active|1|1|category_code|BTREE|
|idx_cat_active|1|2|is_active|BTREE|

### ai_categories
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|code|0|1|code|BTREE|

### ai_category_terms
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|uq_cat_term|0|1|category_code|BTREE|
|uq_cat_term|0|2|term_type|BTREE|
|uq_cat_term|0|3|term|BTREE|
|idx_cat|1|1|category_code|BTREE|

### ai_config_audits
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|idx_target|1|1|target|BTREE|

### ai_prompt_snippets
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|snippet_key|0|1|snippet_key|BTREE|

### ai_settings
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|setting_key|0|1|setting_key|BTREE|

### ai_tone_policies
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|uq_channel_key|0|1|channel|BTREE|
|uq_channel_key|0|2|key_name|BTREE|

### audit_logs
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|

### failed_logins
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|idx_user_ts|1|1|user_ref|BTREE|
|idx_user_ts|1|2|ts|BTREE|

### fee_items
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|code|0|1|code|BTREE|

### fee_packages
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|code|0|1|code|BTREE|

### hq_chat_links
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|thread_id|1|1|thread_id|BTREE|

### hq_chat_messages
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|thread_id|1|1|thread_id|BTREE|

### hq_chat_threads
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|
|idx_hq_chat_threads_shop_updated|1|1|shop_id|BTREE|
|idx_hq_chat_threads_shop_updated|1|2|updated_at|BTREE|

### hq_meetings
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|slot_id|1|1|slot_id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### hq_meeting_slots
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|slot_start|1|1|slot_start|BTREE|
|status|1|1|status|BTREE|
|idx_meeting_slots_status_start|1|1|status|BTREE|
|idx_meeting_slots_status_start|1|2|slot_start|BTREE|

### intents
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|

### invoices
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|owner_id|1|1|owner_id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### invoice_items
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|invoice_id|1|1|invoice_id|BTREE|

### jwt_revocations
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|jti|BTREE|
|idx_exp|1|1|exp_at|BTREE|

### owners
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|owner_code|0|1|owner_code|BTREE|

### owner_contacts
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|owner_id|1|1|owner_id|BTREE|

### package_components
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|package_id|1|1|package_id|BTREE|
|fee_item_id|1|1|fee_item_id|BTREE|

### portal_users
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|login_id|0|1|login_id|BTREE|

### rate_limits
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|key_hash|BTREE|
|PRIMARY|0|2|window_start|BTREE|

### request_templates
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|template_code|0|1|template_code|BTREE|

### sales_records
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|merchant_public_id|1|1|merchant_public_id|BTREE|
|transaction_datetime|1|1|transaction_datetime|BTREE|
|idx_sales_shop_dt|1|1|shop_id|BTREE|
|idx_sales_shop_dt|1|2|transaction_datetime|BTREE|

### shops
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|code|0|1|code|BTREE|
|owner_id|1|1|owner_id|BTREE|

### shop_fee_addons
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### shop_fee_package_assignments
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|
|package_id|1|1|package_id|BTREE|

### shop_options
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### shop_requests
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### shop_royalty_history
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### shop_secrets
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|uniq_shop|0|1|shop_id|BTREE|

### shop_supplies
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|shop_id|1|1|shop_id|BTREE|

### tickets
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|idx_tickets_shop_status_updated|1|1|shop_id|BTREE|
|idx_tickets_shop_status_updated|1|2|status|BTREE|
|idx_tickets_shop_status_updated|1|3|updated_at|BTREE|

### ticket_messages
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|ticket_id|1|1|ticket_id|BTREE|

### tutorial_feedbacks
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|log_id|1|1|log_id|BTREE|

### tutorial_logs
| Key name | Non-unique | Seq | Column | Type |
|---|---|---|---|---|
|PRIMARY|0|1|id|BTREE|
|idx_tutorial_logs_channel_created|1|1|channel|BTREE|
|idx_tutorial_logs_channel_created|1|2|created_at|BTREE|
