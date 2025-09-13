if 'data_loader' not in globals():
    from mage_ai.data_preparation.decorators import data_loader
import requests
from mage_ai.data_preparation.shared.secrets import get_secret_value

@data_loader
def load_customers(*args, **kwargs):
    """
    Carga clientes desde QuickBooks API usando OAuth 2.0.
    """
    # 1. Obtener credenciales desde Secrets
    client_id = get_secret_value("qb_client_id")
    client_secret = get_secret_value("qb_client_secret")
    refresh_token = get_secret_value("qb_refresh_token")
    realm_id = get_secret_value("qb_realm_id")

    # 2. Obtener un access token nuevo desde el refresh token
    auth = (client_id, client_secret)
    resp = requests.post(
        "https://oauth.platform.intuit.com/oauth2/v1/tokens/bearer",
        auth=auth,
        headers={"Accept": "application/json", "Content-Type": "application/x-www-form-urlencoded"},
        data={"grant_type": "refresh_token", "refresh_token": refresh_token}
    )
    access_token = resp.json()["access_token"]

    # 3. Llamar a la API de QuickBooks para obtener Customers
    url = f"https://sandbox-quickbooks.api.intuit.com/v3/company/{realm_id}/query"
    query = "select * from Customer"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Accept": "application/json",
        "Content-Type": "application/text"
    }
    response = requests.post(url, headers=headers, data=query)

    data = response.json()
    return data
