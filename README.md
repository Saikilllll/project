import requests
from google_auth_oauthlib.flow import InstalledAppFlow
from msal import PublicClientApplication
from requests_oauthlib import OAuth1Session

class GoogleCalendarAgent:
    def __init__(self, token):
        self.token = token
        self.base_url = "https://www.googleapis.com/calendar/v3"

    def get_events(self):
        url = f"{self.base_url}/calendars/primary/events"
        headers = {
            "Authorization": f"Bearer {self.token}"
        }
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            return response.json().get("items", [])
        else:
            raise Exception(f"Error fetching events: {response.status_code}")

class OutlookCalendarAgent:
    def __init__(self, token):
        self.token = token
        self.base_url = "https://graph.microsoft.com/v1.0"

    def get_events(self):
        url = f"{self.base_url}/me/events"
        headers = {
            "Authorization": f"Bearer {self.token}"
        }
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            return response.json().get("value", [])
        else:
            raise Exception(f"Error fetching events: {response.status_code}")

class SocialMediaAgent:
    def __init__(self, platform_name, token, api_url):
        self.platform_name = platform_name
        self.token = token
        self.api_url = api_url

    def get_updates(self):
        headers = {
            "Authorization": f"Bearer {self.token}"
        }
        response = requests.get(self.api_url, headers=headers)
        if response.status_code == 200:
            return {"platform": self.platform_name, "updates": response.json()}
        else:
            raise Exception(f"Error fetching updates from {self.platform_name}: {response.status_code}")

class UnifiedIntegration:
    def __init__(self):
        self.agents = []

    def add_agent(self, agent):
        self.agents.append(agent)

    def get_combined_data(self):
        combined_data = {
            "events": [],
            "social_updates": []
        }
        for agent in self.agents:
            try:
                if isinstance(agent, (GoogleCalendarAgent, OutlookCalendarAgent)):
                    combined_data["events"].extend(agent.get_events())
                elif isinstance(agent, SocialMediaAgent):
                    combined_data["social_updates"].append(agent.get_updates())
            except Exception as e:
                print(f"Error fetching data from agent: {e}")
        return combined_data

# Helper Functions for OAuth
def get_google_token():
    SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']
    flow = InstalledAppFlow.from_client_secrets_file('client_secrets.json', SCOPES)
    credentials = flow.run_local_server(port=0)
    return credentials.token

def get_outlook_token():
    CLIENT_ID = 'YOUR_OUTLOOK_CLIENT_ID'
    AUTHORITY = 'https://login.microsoftonline.com/common'
    SCOPES = ['https://graph.microsoft.com/Calendars.Read']

    app = PublicClientApplication(CLIENT_ID, authority=AUTHORITY)
    result = app.acquire_token_interactive(SCOPES)
    return result['access_token']

def get_facebook_token(code):
    APP_ID = "YOUR_FACEBOOK_APP_ID"
    APP_SECRET = "YOUR_FACEBOOK_APP_SECRET"
    REDIRECT_URI = "YOUR_FACEBOOK_REDIRECT_URI"

    url = f"https://graph.facebook.com/v12.0/oauth/access_token"
    params = {
        "client_id": APP_ID,
        "redirect_uri": REDIRECT_URI,
        "client_secret": APP_SECRET,
        "code": code
    }
    response = requests.get(url, params=params)
    return response.json()['access_token']

def get_twitter_token():
    API_KEY = "YOUR_TWITTER_API_KEY"
    API_SECRET_KEY = "YOUR_TWITTER_API_SECRET_KEY"

    request_token_url = "https://api.twitter.com/oauth/request_token"
    oauth = OAuth1Session(API_KEY, client_secret=API_SECRET_KEY)
    fetch_response = oauth.fetch_request_token(request_token_url)
    return fetch_response

# Main Integration
if __name__ == "__main__":
    # Retrieve tokens (replace placeholders with real data)
    try:
        google_token = get_google_token()
        outlook_token = get_outlook_token()
        facebook_token = "YOUR_FACEBOOK_ACCESS_TOKEN"  # Replace after retrieving Facebook token
        twitter_token = "YOUR_TWITTER_ACCESS_TOKEN"    # Replace after retrieving Twitter token

        # Create agents
        google_agent = GoogleCalendarAgent(google_token)
        outlook_agent = OutlookCalendarAgent(outlook_token)
        facebook_agent = SocialMediaAgent("Facebook", facebook_token, "https://graph.facebook.com/v12.0/me/feed")
        twitter_agent = SocialMediaAgent("Twitter", twitter_token, "https://api.twitter.com/2/users/me/tweets")

        # Unified integration
        unified_integration = UnifiedIntegration()
        unified_integration.add_agent(google_agent)
        unified_integration.add_agent(outlook_agent)
        unified_integration.add_agent(facebook_agent)
        unified_integration.add_agent(twitter_agent)

        # Fetch combined data
        combined_data = unified_integration.get_combined_data()

        # Print combined data
        print("Combined Calendar Events:")
        for event in combined_data["events"]:
            print(event)

        print("\nCombined Social Media Updates:")
        for update in combined_data["social_updates"]:
            print(update)

    except Exception as e:
        print(f"Error during integration: {e}")
