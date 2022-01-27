# workloadmeta

## workload

A workload is any unit or work being done by a piece of software, like a process, a container, a kubernetes pod, or a task in any cloud provider. 



## Store

### What

Store is a central storage of metadata about workloads.

```go
type Store interface {
	Start(ctx context.Context)
	Subscribe(name string, priority SubscriberPriority, filter *Filter) chan EventBundle
	Unsubscribe(ch chan EventBundle)
	GetContainer(id string) (*Container, error)
	ListContainers() ([]*Container, error)
	GetKubernetesPod(id string) (*KubernetesPod, error)
	GetKubernetesPodForContainer(containerID string) (*KubernetesPod, error)
	GetECSTask(id string) (*ECSTask, error)
	Notify(events []CollectorEvent)
	Dump(verbose bool) WorkloadDumpResponse
}
```



### Where

```go
// GetGlobalStore returns a global instance of the workloadmeta store,
// creating one if it doesn't exist. Start() needs to be called before any data
// collection happens.
func GetGlobalStore() Store {
	initOnce.Do(func() {
		if globalStore == nil {
			globalStore = NewStore(collectorCatalog)
		}
	})

	return globalStore
}
```

```go
// LoadComponents configures several common Agent components:
// tagger, collector, scheduler and autodiscovery
func LoadComponents(confdPath string) {
	if flavor.GetFlavor() != flavor.ClusterAgent {
		workloadmeta.GetGlobalStore().Start(context.Background())

		// start the tagger. must be done before autodiscovery, as it needs to
		// be the first subscribed to metadata store to avoid race conditions.
		tagger.SetDefaultTagger(local.NewTagger(collectors.DefaultCatalog))
		if err := tagger.Init(); err != nil {
			log.Errorf("failed to start the tagger: %s", err)
		}
	}

	// create the Collector instance and start all the components
	// NOTICE: this will also setup the Python environment, if available
	Coll = collector.NewCollector(GetPythonPaths()...)

	// creating the meta scheduler
	metaScheduler := scheduler.NewMetaScheduler()

	// registering the check scheduler
	metaScheduler.Register("check", collector.InitCheckScheduler(Coll))

	// registering the logs scheduler
	if lstatus.Get().IsRunning {
		metaScheduler.Register("logs", lsched.GetScheduler())
	}

	// setup autodiscovery
	confSearchPaths := []string{
		confdPath,
		filepath.Join(GetDistPath(), "conf.d"),
		"",
	}

	// setup autodiscovery. must be done after the tagger is initialized
	// because of subscription to metadata store.
	AC = setupAutoDiscovery(confSearchPaths, metaScheduler)
}
```



- cmd/agent/common
  - [workloadmeta.GetGlobalStore().Start(context.Background())](https://github.com/DataDog/datadog-agent/blob/dfadf68bf9f718de2b0b0864546c71ac07d88069/cmd/agent/common/loader.go#L31)
    - agent
    - cluster-agent
    - cluster-agent-cloudfoundry
    - check
    - jmx
- dogstatsd
  - [workloadmeta.GetGlobalStore().Start(context.Background())]()
- process-agent
  - [workloadmeta.GetGlobalStore().Start(context.Background())](https://github.com/DataDog/datadog-agent/blob/dfadf68bf9f718de2b0b0864546c71ac07d88069/cmd/dogstatsd/main.go#L217)
- trace-agent
  - [workloadmeta.GetGlobalStore().Start(context.Background())](https://github.com/DataDog/datadog-agent/blob/dfadf68bf9f718de2b0b0864546c71ac07d88069/pkg/trace/agent/run.go#L160)

https://github.com/DataDog/datadog-agent/blob/dfadf68bf9f718de2b0b0864546c71ac07d88069/releasenotes/notes/remote-tagger-81fd36a9e0ff39e8.yaml

The core agent now exposes a gRPC API to expose tags to the other agents.
The following settings are now introduced to allow each of the agents to use this API(they all default to false):
- apm_config.remote_tagger
- logs_config.remote_tagger
- process_config.remote_tagger
- security_agent.remote_tagger
- runtime_security_agent.remote_tagger



### How

```go
// Start starts the workload metadata store.
func (s *store) Start(ctx context.Context) {
	go func() {
		health := health.RegisterLiveness("workloadmeta-store")
		for {
			select {
			case <-health.C:

			case evs := <-s.eventCh:
				s.handleEvents(evs)

			case <-ctx.Done():
				err := health.Deregister()
				if err != nil {
					log.Warnf("error de-registering health check: %s", err)
				}

				return
			}
		}
	}()

	go func() {
		retryTicker := time.NewTicker(retryCollectorInterval)
		pullTicker := time.NewTicker(pullCollectorInterval)
		health := health.RegisterLiveness("workloadmeta-puller")
		pullCtx, pullCancel := context.WithTimeout(ctx, pullCollectorInterval)

		// Start a pull immediately to fill the store without waiting for the
		// next tick.
		s.pull(pullCtx)

		for {
			select {
			case <-health.C:

			case <-pullTicker.C:
				// pullCtx will always be expired at this point
				// if pullTicker has the same duration as
				// pullCtx, so we cancel just as good practice
				pullCancel()

				pullCtx, pullCancel = context.WithTimeout(ctx, pullCollectorInterval)
				s.pull(pullCtx)

			case <-retryTicker.C:
				stop := s.startCandidates(ctx)

				if stop {
					retryTicker.Stop()
				}

			case <-ctx.Done():
				retryTicker.Stop()
				pullTicker.Stop()

				pullCancel()

				err := health.Deregister()
				if err != nil {
					log.Warnf("error de-registering health check: %s", err)
				}

				return
			}
		}
	}()

	s.startCandidates(ctx)

	log.Info("workloadmeta store initialized successfully")
}
```

```go
func (s *store) startCandidates(ctx context.Context) bool {
	s.collectorMut.Lock()
	defer s.collectorMut.Unlock()

	for id, c := range s.candidates {
		err := c.Start(ctx, s)

		// Leave candidates that returned a retriable error to be
		// re-started in the next tick
		if err != nil && retry.IsErrWillRetry(err) {
			log.Debugf("workloadmeta collector %q could not start, but will retry. error: %s", id, err)
			continue
		}

		// Store successfully started collectors for future reference
		if err == nil {
			log.Infof("workloadmeta collector %q started successfully", id)
			s.collectors[id] = c
		} else {
			log.Infof("workloadmeta collector %q could not start. error: %s", id, err)
		}

		// Remove non-retriable and successfully started collectors
		// from the list of candidates so they're not retried in the
		// next tick
		delete(s.candidates, id)
	}

	return len(s.candidates) == 0
}

func (s *store) pull(ctx context.Context) {
	s.collectorMut.RLock()
	defer s.collectorMut.RUnlock()

	for id, c := range s.collectors {
		// Run each pull in its own separate goroutine to reduce
		// latency and unlock the main goroutine to do other work.
		go func(id string, c Collector) {
			err := c.Pull(ctx)
			if err != nil {
				log.Warnf("error pulling from collector %q: %s", id, err.Error())
				telemetry.PullErrors.Inc(id)
			}
		}(id, c)
	}
}
```



## Collectors

- [cloudfoundry](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/cloudfoundry)
- [containerd](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/containerd)
- [docker](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/docker)
- [ecs](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/ecs)
- [ecsfargate](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/ecsfargate)
- [kubelet](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/kubelet)
- [kubemetadata](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/kubemetadata)
- [podman](https://pkg.go.dev/github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/podman)

```go
package collectors

import (
	// this package only loads the collectors
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/cloudfoundry"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/containerd"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/docker"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/ecs"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/ecsfargate"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/kubelet"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/kubemetadata"
	_ "github.com/DataDog/datadog-agent/pkg/workloadmeta/collectors/podman"
)
```

```go
// Collector is responsible for collecting metadata about workloads.
type Collector interface {
	// Start starts a collector. The collector should run until the context
	// is done. It also gets a reference to the store that started it so it
	// can use Notify, or get access to other entities in the store.
	Start(context.Context, Store) error

	// Pull triggers an entity collection. To be used by collectors that
	// don't have streaming functionality, and called periodically by the
	// store.
	Pull(context.Context) error
}

type collectorFactory func() Collector

var collectorCatalog = make(map[string]collectorFactory)

// RegisterCollector registers a new collector, identified by an id for logging
// and telemetry purposes, to be used by the store.
func RegisterCollector(id string, c collectorFactory) {
	collectorCatalog[id] = c
}
```



### containerd

GRPC API,  watch containerd event

### docker

GRPC API, watch docker events

### podman

RESTful API, get containers

### kubelet

watch local pods

### kubemetadata

- node-level  
  - access kubelet api, get local pods
- cluster-level
  - cluster-agent enable
    - cluster-agent access kubernetes api
    - node-agent access cluster api
  - cluster-agent disable
    - node-agent access kubernetes api



