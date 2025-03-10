package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"strconv"
	"strings"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// Command line flags
var (
	listenAddress   = flag.String("web.listen-address", ":9734", "Address on which to expose metrics")
	metricsPath     = flag.String("web.telemetry-path", "/metrics", "Path under which to expose metrics")
	audiobookshelfURL = flag.String("audiobookshelf.url", "", "Audiobookshelf URL (e.g., http://localhost:8000)")
	apiToken        = flag.String("audiobookshelf.token", "", "Audiobookshelf API token")
	scrapeTimeout   = flag.Duration("scrape.timeout", 10*time.Second, "Timeout for API requests")
)

// AudiobookshelfExporter represents the exporter for Audiobookshelf
type AudiobookshelfExporter struct {
	url      string
	token    string
	client   *http.Client
	timeout  time.Duration

	// Metrics
	up                     prometheus.Gauge
	totalScrapes           prometheus.Counter
	scrapeErrors           *prometheus.CounterVec
	scrapeDuration         prometheus.Gauge
	libraryInfo            *prometheus.GaugeVec
	totalItems             *prometheus.GaugeVec
	storageUsage           prometheus.Gauge
	totalSpace             prometheus.Gauge
	activeSessions         prometheus.Gauge
	totalUsers             prometheus.Gauge
	recentlyAddedItems     prometheus.Gauge
	recentlyUpdatedItems   prometheus.Gauge
	libraryItemsByMediaType *prometheus.GaugeVec
	podcastEpisodes        *prometheus.GaugeVec
}

// NewAudiobookshelfExporter creates a new Audiobookshelf exporter
func NewAudiobookshelfExporter(url, token string, timeout time.Duration) *AudiobookshelfExporter {
	return &AudiobookshelfExporter{
		url:     url,
		token:   token,
		client:  &http.Client{Timeout: timeout},
		timeout: timeout,

		up: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "up",
			Help:      "Was the last scrape of Audiobookshelf successful",
		}),

		totalScrapes: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: "audiobookshelf",
			Name:      "exporter_scrapes_total",
			Help:      "Current total Audiobookshelf scrapes",
		}),

		scrapeErrors: prometheus.NewCounterVec(prometheus.CounterOpts{
			Namespace: "audiobookshelf",
			Name:      "exporter_scrape_errors_total",
			Help:      "Current total Audiobookshelf scrape errors",
		}, []string{"collector"}),

		scrapeDuration: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "exporter_scrape_duration_seconds",
			Help:      "Duration of the scrape",
		}),

		libraryInfo: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "library_info",
			Help:      "Information about Audiobookshelf libraries",
		}, []string{"library_id", "library_name", "library_type"}),

		totalItems: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "items_total",
			Help:      "Total number of items in libraries",
		}, []string{"library_id", "library_name"}),

		storageUsage: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "storage_bytes_used",
			Help:      "Storage space used by Audiobookshelf in bytes",
		}),

		totalSpace: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "storage_bytes_total",
			Help:      "Total storage space available to Audiobookshelf in bytes",
		}),

		activeSessions: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "active_sessions",
			Help:      "Number of active listening sessions",
		}),

		totalUsers: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "users_total",
			Help:      "Total number of users",
		}),

		recentlyAddedItems: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "recently_added_items",
			Help:      "Number of items added in the last 24 hours",
		}),

		recentlyUpdatedItems: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "recently_updated_items",
			Help:      "Number of items updated in the last 24 hours",
		}),

		libraryItemsByMediaType: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "library_items_by_media_type",
			Help:      "Number of library items by media type",
		}, []string{"library_id", "media_type"}),

		podcastEpisodes: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: "audiobookshelf",
			Name:      "podcast_episodes",
			Help:      "Number of podcast episodes by status",
		}, []string{"library_id", "status"}),
	}
}

// Describe implements the prometheus.Collector interface
func (e *AudiobookshelfExporter) Describe(ch chan<- *prometheus.Desc) {
	e.up.Describe(ch)
	e.totalScrapes.Describe(ch)
	e.scrapeErrors.Describe(ch)
	e.scrapeDuration.Describe(ch)
	e.libraryInfo.Describe(ch)
	e.totalItems.Describe(ch)
	e.storageUsage.Describe(ch)
	e.totalSpace.Describe(ch)
	e.activeSessions.Describe(ch)
	e.totalUsers.Describe(ch)
	e.recentlyAddedItems.Describe(ch)
	e.recentlyUpdatedItems.Describe(ch)
	e.libraryItemsByMediaType.Describe(ch)
	e.podcastEpisodes.Describe(ch)
}

// Collect implements the prometheus.Collector interface
func (e *AudiobookshelfExporter) Collect(ch chan<- prometheus.Metric) {
	e.totalScrapes.Inc()
	
	scrapeStart := time.Now()
	defer func() {
		e.scrapeDuration.Set(time.Since(scrapeStart).Seconds())
		e.scrapeDuration.Collect(ch)
	}()

	if err := e.scrape(); err != nil {
		log.Printf("Error scraping Audiobookshelf: %s", err)
		e.up.Set(0)
	} else {
		e.up.Set(1)
	}

	e.up.Collect(ch)
	e.totalScrapes.Collect(ch)
	e.scrapeErrors.Collect(ch)
	e.libraryInfo.Collect(ch)
	e.totalItems.Collect(ch)
	e.storageUsage.Collect(ch)
	e.totalSpace.Collect(ch)
	e.activeSessions.Collect(ch)
	e.totalUsers.Collect(ch)
	e.recentlyAddedItems.Collect(ch)
	e.recentlyUpdatedItems.Collect(ch)
	e.libraryItemsByMediaType.Collect(ch)
	e.podcastEpisodes.Collect(ch)
}

// scrape collects data from the Audiobookshelf API
func (e *AudiobookshelfExporter) scrape() error {
	if err := e.collectServerStats(); err != nil {
		e.scrapeErrors.WithLabelValues("server_stats").Inc()
		return err
	}

	if err := e.collectLibraries(); err != nil {
		e.scrapeErrors.WithLabelValues("libraries").Inc()
		return err
	}

	if err := e.collectUsers(); err != nil {
		e.scrapeErrors.WithLabelValues("users").Inc()
		return err
	}

	if err := e.collectSessions(); err != nil {
		e.scrapeErrors.WithLabelValues("sessions").Inc()
		return err
	}

	return nil
}

// apiRequest makes a request to the Audiobookshelf API
func (e *AudiobookshelfExporter) apiRequest(path string) (map[string]interface{}, error) {
	baseURL, err := url.Parse(e.url)
	if err != nil {
		return nil, fmt.Errorf("invalid API URL: %v", err)
	}

	// Join base URL with path
	endpoint, err := url.Parse(path)
	if err != nil {
		return nil, fmt.Errorf("invalid API endpoint: %v", err)
	}
	
	fullURL := baseURL.ResolveReference(endpoint)

	req, err := http.NewRequest("GET", fullURL.String(), nil)
	if err != nil {
		return nil, fmt.Errorf("failed to create request: %v", err)
	}

	// Add Authorization header
	req.Header.Add("Authorization", "Bearer "+e.token)

	resp, err := e.client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("API request failed: %v", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("API returned non-200 status code: %d", resp.StatusCode)
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("failed to read response body: %v", err)
	}

	var result map[string]interface{}
	if err := json.Unmarshal(body, &result); err != nil {
		return nil, fmt.Errorf("failed to parse JSON response: %v", err)
	}

	return result, nil
}

// collectServerStats collects stats about the server
func (e *AudiobookshelfExporter) collectServerStats() error {
	data, err := e.apiRequest("/api/server-stats")
	if err != nil {
		return err
	}

	// Extract storage information
	if storageInfo, ok := data["storageSize"].(map[string]interface{}); ok {
		// Convert to bytes - API returns size in bytes
		if usedStr, ok := storageInfo["size"].(float64); ok {
			e.storageUsage.Set(usedStr)
		}
		
		if totalStr, ok := storageInfo["total"].(float64); ok {
			e.totalSpace.Set(totalStr)
		}
	}

	// Check for recently added or updated items
	oneDayAgo := time.Now().Add(-24 * time.Hour).UnixNano() / int64(time.Millisecond)
	
	var recentlyAdded, recentlyUpdated int

	if recentItems, ok := data["recentlyAdded"].([]interface{}); ok {
		for _, item := range recentItems {
			if itemMap, ok := item.(map[string]interface{}); ok {
				if addedAt, ok := itemMap["addedAt"].(float64); ok {
					if int64(addedAt) > oneDayAgo {
						recentlyAdded++
					}
				}
			}
		}
	}

	if recentItems, ok := data["recentlyUpdated"].([]interface{}); ok {
		for _, item := range recentItems {
			if itemMap, ok := item.(map[string]interface{}); ok {
				if updatedAt, ok := itemMap["updatedAt"].(float64); ok {
					if int64(updatedAt) > oneDayAgo {
						recentlyUpdated++
					}
				}
			}
		}
	}
	
	e.recentlyAddedItems.Set(float64(recentlyAdded))
	e.recentlyUpdatedItems.Set(float64(recentlyUpdated))

	return nil
}

// collectLibraries collects information about libraries
func (e *AudiobookshelfExporter) collectLibraries() error {
	data, err := e.apiRequest("/api/libraries")
	if err != nil {
		return err
	}

	if libraries, ok := data["libraries"].([]interface{}); ok {
		for _, lib := range libraries {
			if library, ok := lib.(map[string]interface{}); ok {
				libraryID, _ := library["id"].(string)
				libraryName, _ := library["name"].(string)
				libraryType, _ := library["mediaType"].(string)
				
				// Set library info metric
				e.libraryInfo.WithLabelValues(libraryID, libraryName, libraryType).Set(1)
				
				// Get details for this specific library
				libDetails, err := e.apiRequest("/api/libraries/" + libraryID)
				if err != nil {
					continue
				}
				
				// Get total items
				if numItems, ok := libDetails["mediaCount"].(float64); ok {
					e.totalItems.WithLabelValues(libraryID, libraryName).Set(numItems)
				}
				
				// Get media type breakdown
				if mediaTypes, ok := libDetails["media"].(map[string]interface{}); ok {
					for mediaType, count := range mediaTypes {
						if countVal, ok := count.(float64); ok {
							e.libraryItemsByMediaType.WithLabelValues(libraryID, mediaType).Set(countVal)
						}
					}
				}
				
				// For podcast libraries, get episode stats
				if libraryType == "podcast" {
					var played, unplayed, inProgress float64
					
					if episodes, ok := libDetails["episodes"].([]interface{}); ok {
						for _, episode := range episodes {
							if ep, ok := episode.(map[string]interface{}); ok {
								if status, ok := ep["status"].(string); ok {
									switch status {
									case "played":
										played++
									case "unplayed":
										unplayed++
									case "in-progress":
										inProgress++
									}
								}
							}
						}
					}
					
					e.podcastEpisodes.WithLabelValues(libraryID, "played").Set(played)
					e.podcastEpisodes.WithLabelValues(libraryID, "unplayed").Set(unplayed)
					e.podcastEpisodes.WithLabelValues(libraryID, "in_progress").Set(inProgress)
				}
			}
		}
	}
	
	return nil
}

// collectUsers collects user information
func (e *AudiobookshelfExporter) collectUsers() error {
	data, err := e.apiRequest("/api/users")
	if err != nil {
		return err
	}

	if users, ok := data["users"].([]interface{}); ok {
		e.totalUsers.Set(float64(len(users)))
	}
	
	return nil
}

// collectSessions collects information about active sessions
func (e *AudiobookshelfExporter) collectSessions() error {
	data, err := e.apiRequest("/api/sessions")
	if err != nil {
		return err
	}

	if sessions, ok := data["sessions"].([]interface{}); ok {
		activeCount := 0
		for _, session := range sessions {
			if sess, ok := session.(map[string]interface{}); ok {
				if isActive, ok := sess["isActive"].(bool); ok && isActive {
					activeCount++
				}
			}
		}
		e.activeSessions.Set(float64(activeCount))
	}
	
	return nil
}

func main() {
	flag.Parse()

	if *audiobookshelfURL == "" {
		log.Fatal("Audiobookshelf URL is required")
	}

	if *apiToken == "" {
		log.Fatal("Audiobookshelf API token is required")
	}

	// Create and register exporter
	exporter := NewAudiobookshelfExporter(*audiobookshelfURL, *apiToken, *scrapeTimeout)
	prometheus.MustRegister(exporter)

	// Create HTTP server
	http.Handle(*metricsPath, promhttp.Handler())
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`<html>
			<head><title>Audiobookshelf Exporter</title></head>
			<body>
			<h1>Audiobookshelf Exporter</h1>
			<p><a href="` + *metricsPath + `">Metrics</a></p>
			</body>
			</html>`))
	})

	log.Printf("Starting Audiobookshelf exporter on %s", *listenAddress)
	log.Fatal(http.ListenAndServe(*listenAddress, nil))
}
