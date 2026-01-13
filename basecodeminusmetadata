import json
import boto3
import urllib3
from base64 import b64encode

http = urllib3.PoolManager()

secrets = boto3.client("secretsmanager")
secret = json.loads(
    secrets.get_secret_value(SecretId="user")["SecretString"]
)

SNOW_INSTANCE = secret["SNOW_INSTANCE"]
SNOW_USER = secret["SNOW_USER"]
SNOW_PASSWORD = secret["SNOW_PASSWORD"]

AUTH = b64encode(f"{SNOW_USER}:{SNOW_PASSWORD}".encode()).decode()
HEADERS = {
    "Content-Type": "application/json",
    "Accept": "application/json",
    "Authorization": f"Basic {AUTH}"
}

NORMAL_CHANGE_MODEL_SYS_ID = "your_actual_sys_id_here"  # Replace with real sys_id

def handler(event, context):
    try:
        event_arn = event.get("detail", {}).get("eventArn", "TEST_EVENT_ARN")
        
        # 1. Create Case (not incident)
        case_payload = {
            "short_description": "AWS Health EC2 Event",
            "description": "Automatically created from AWS Health",
            "priority": "4",
            "correlation_id": event_arn
        }
        
        print("Creating case...")
        case_response = http.request(
            'POST',
            f"{SNOW_INSTANCE}/api/now/table/sn_customerservice_case",
            headers=HEADERS,
            body=json.dumps(case_payload)
        )
        
        print(f"Case response status: {case_response.status}")
        print(f"Case response body: {case_response.data.decode('utf-8')}")
        
        case_data = json.loads(case_response.data.decode('utf-8'))
        
        if case_response.status != 201:
            return {
                "error": "Failed to create case",
                "status": case_response.status,
                "response": case_data
            }
        
        case_sys_id = case_data["result"]["sys_id"]
        print(f"Case created: {case_sys_id}")
        
        # 2. Create Change
        change_payload = {
            "type": "normal",
            "chg_model": NORMAL_CHANGE_MODEL_SYS_ID,
            "risk": "medium",
            "short_description": "AWS Health EC2 Event",
            "description": "Automatically created from AWS Health",
            "correlation_id": event_arn
        }
        
        print("Creating change request...")
        chg_response = http.request(
            'POST',
            f"{SNOW_INSTANCE}/api/now/table/change_request",
            headers=HEADERS,
            body=json.dumps(change_payload)
        )
        
        print(f"Change response status: {chg_response.status}")
        print(f"Change response body: {chg_response.data.decode('utf-8')}")
        
        chg_data = json.loads(chg_response.data.decode('utf-8'))
        
        if chg_response.status != 201:
            return {
                "error": "Failed to create change",
                "status": chg_response.status,
                "response": chg_data
            }
        
        change_sys_id = chg_data["result"]["sys_id"]
        print(f"Change created: {change_sys_id}")
        
        # 3. Link Case → Change (if your ServiceNow supports this relationship)
        print("Linking case to change...")
        link_response = http.request(
            'PATCH',
            f"{SNOW_INSTANCE}/api/now/table/sn_customerservice_case/{case_sys_id}",
            headers=HEADERS,
            body=json.dumps({"u_related_change": change_sys_id})  # Field name might differ
        )
        
        print(f"Link response status: {link_response.status}")
        
        return {
            "status": "ok",
            "case_sys_id": case_sys_id,
            "change_sys_id": change_sys_id
        }
        
    except Exception as e:
        print(f"ERROR: {str(e)}")
        import traceback
        print(traceback.format_exc())
        return {
            "error": str(e),
            "traceback": traceback.format_exc()
        }
