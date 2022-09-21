# discovery

## Target Group

```go
// Group is a set of targets with a common label set(production , test, staging etc.).
type Group struct {
	// Targets is a list of targets identified by a label set. Each target is
	// uniquely identifiable in the group by its address label.
	Targets []model.LabelSet
	// Labels is a set of labels that is common across all targets in the group.
	Labels model.LabelSet

	// Source is an identifier that describes a group of targets.
	Source string
}
```



```go
// A StaticConfig is a Config that provides a static list of targets.
type StaticConfig []*targetgroup.Group
```



```go
// Discoverer provides information about target groups. It maintains a set
// of sources from which TargetGroups can originate. Whenever a discovery provider
// detects a potential change, it sends the TargetGroup through its channel.
//
// Discoverer does not know if an actual change happened.
// It does guarantee that it sends the new TargetGroup whenever a change happens.
//
// Discoverers should initially send a full set of all discoverable TargetGroups.
type Discoverer interface {
	// Run hands a channel to the discovery provider (Consul, DNS, etc.) through which
	// it can send updated target groups. It must return when the context is canceled.
	// It should not close the update channel on returning.
	Run(ctx context.Context, up chan<- []*targetgroup.Group)
}

```



## Discovery



```go
// discoveryManager interfaces the discovery manager. This is used to keep using
// the manager that restarts SD's on reload for a few releases until we feel
// the new manager can be enabled for all users.
type discoveryManager interface {
	ApplyConfig(cfg map[string]discovery.Configs) error
	Run() error
	SyncCh() <-chan map[string][]*targetgroup.Group
}

// Manager maintains a set of discovery providers and sends each update to a map channel.
// Targets are grouped by the target set name.
type Manager struct {
	logger   log.Logger
	name     string
	httpOpts []config.HTTPClientOption
	mtx      sync.RWMutex
	ctx      context.Context

	// Some Discoverers(e.g. k8s) send only the updates for a given target group,
	// so we use map[tg.Source]*targetgroup.Group to know which group to update.
	targets    map[poolKey]map[string]*targetgroup.Group
	targetsMtx sync.Mutex

	// providers keeps track of SD providers.
	providers []*Provider
	// The sync channel sends the updates as a map where the key is the job value from the scrape config.
	syncCh chan map[string][]*targetgroup.Group

	// How long to wait before sending updates to the channel. The variable
	// should only be modified in unit tests.
	updatert time.Duration

	// The triggerSend channel signals to the Manager that new updates have been received from providers.
	triggerSend chan struct{}

	// lastProvider counts providers registered during Manager's lifetime.
	lastProvider uint
}

// Provider holds a Discoverer instance, its configuration, cancel func and its subscribers.
type Provider struct {
	name   string
	d      Discoverer
	config interface{}

	cancel context.CancelFunc
	// done should be called after cleaning up resources associated with cancelled provider.
	done func()

	mu   sync.RWMutex
	subs map[string]struct{}

	// newSubs is used to temporary store subs to be used upon config reload completion.
	newSubs map[string]struct{}
}
```





### Implements

- manager
  - Discoverer
- registry
  - Config
- Discoverer interface
  - consul
  - file
  - kubernetes/endpoints
  - kubernetes/endpointslice
  - kubernetes/ingress
  - kubernetes/kubernetes
  - kubernetes/node
  - kubernetes/pod
  - kubernetes/service
  - refresh
  - xds
  - zookeeper
  - examples/custom-sd
- Config interface
  - aws/ec2
  - aws/lightsail
  - azure
  - consul
  - digitalocean
  - static
  - dns
  - eureka
  - file
  - gce
  - hetzner
  - http
  - ionos
  - kubernetes
  - linode
  - marathon
  - docker
  - dockerswarm
  - nomad
  - openstack
  - puppetdb
  - scaleway
  - triton
  - uyuni
  - vultr
  - kuma
  - zookeeper



## Reference

- https://github.com/prometheus/prometheus/tree/main/discovery
