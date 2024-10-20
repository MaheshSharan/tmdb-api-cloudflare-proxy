# TMDB API Cloudflare Proxy Solution

<p align="center">
  <img src="https://www.themoviedb.org/assets/2/v4/logos/v2/blue_short-8e7b30f73a4020692ccca9c88bafe5dcb6f8a62a4c6bc55cd9ba82bb2cd95f6c.svg" alt="TMDB Logo" height="40">
  
</p>

## Problem Description

Users, especially those in India using mobile networks like Jio and Airtel, may experience issues accessing the TMDB (The Movie Database) API when using mobile data or hotspots. This problem manifests as:

- Consistent failures to fetch data
- 500 status code errors
- Issues related to network restrictions or throttling by mobile providers
- Potential firewall misconfigurations on mobile networks


## Solution Overview

To resolve this issue, we'll use a Cloudflare Worker as a proxy for TMDB API requests. This approach:

- Bypasses potential network restrictions
- Ensures consistent access across different network types

## Step-by-Step Guide

### 1. Set Up a Cloudflare Account

1. Go to [Cloudflare's website](https://www.cloudflare.com/) and sign up for a free account if you don't have one.
2. Log in to the Cloudflare dashboard.

### 2. Create a Cloudflare Worker

1. In the Cloudflare dashboard, click on the "Workers" tab.
2. Click the "Create a Service" button.
3. Choose a name for your worker (e.g., "tmdb-proxy").
4. Select a subdomain for your worker (e.g., "tmdb-proxy.yourdomain.workers.dev").


### 3. Implement the Proxy Code

Replace the default code in the worker editor with the following:

```javascript
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
    const url = new URL(request.url);
    const targetUrl = url.searchParams.get('url');
    if (!targetUrl) {
        return new Response('Missing URL parameter', { status: 400 });
    }
    const response = await fetch(targetUrl, {
        method: request.method,
        headers: {
            ...request.headers,
            'Access-Control-Allow-Origin': '*', // Allow CORS
        },
    });
    return new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: {
            'Access-Control-Allow-Origin': '*', // Allow CORS
            ...response.headers,
        },
    });
}
```

Click "Save and Deploy" to publish your worker.

### 4. Update Your Backend Code

Update your TMDB service to use the Cloudflare Worker. Here's an example using Node.js and axios:

```javascript
const axios = require('axios');

class TMDBService {
  constructor() {
    this.baseURL = 'https://api.themoviedb.org/3';
    this.apiKey = process.env.TMDB_API_KEY;
    this.proxyURL = 'https://your-worker-name.your-subdomain.workers.dev/';

    this.client = axios.create({
      baseURL: this.proxyURL,
    });
  }

  async fetchFromTMDB(endpoint, params = {}) {
    const url = `${this.baseURL}${endpoint}?api_key=${this.apiKey}`;
    const encodedUrl = encodeURIComponent(url);
    try {
      const response = await this.client.get(`?url=${encodedUrl}`, { params });
      return response.data;
    } catch (error) {
      console.error(`Error fetching data from TMDB (endpoint: ${endpoint}):`, error);
      throw error;
    }
  }

  // Implement other methods (fetchPopularMovies, fetchTVShowDetails, etc.) using fetchFromTMDB
}
```

### 5. Update Your Frontend Code

If you're making API calls directly from the frontend, update your API service to use your backend endpoint:

```javascript
const API_BASE_URL = 'http://your-backend-url.com/api';

const fetchWithRetry = async (url, options = {}, maxRetries = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) throw new Error('Network response was not ok');
      return await response.json();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1))); // Exponential backoff
    }
  }
};

export const fetchPopularMovies = () => fetchWithRetry(`${API_BASE_URL}/movies/popular`);
export const fetchTVShowDetails = (tvShowId) => fetchWithRetry(`${API_BASE_URL}/tv/${tvShowId}`);
// Implement other API methods similarly
```

### 6. Test Your Implementation

Test your application using various network conditions:
- Wi-Fi
- Mobile data
- Hotspot

Ensure it's working consistently across all network types.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| CORS errors | Ensure your Cloudflare Worker is properly configured to handle CORS as shown in the worker code above. |
| API key issues | Check your TMDB API key is correctly set in your environment variables. |
| Rate limiting | Monitor your Cloudflare Worker usage. If you start hitting rate limits, consider implementing caching strategies. |

## Contributing

We welcome contributions to improve this solution:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details.

## Acknowledgments

- [TMDB](https://www.themoviedb.org/) for their excellent API
- [Cloudflare](https://www.cloudflare.com/) for providing the Workers platform
