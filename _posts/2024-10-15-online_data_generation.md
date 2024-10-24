---
layout: post
title: Online Data Generation for Better, Faster, and Cheaper Training
date: 2024-10-15 
description: An online training framework design for AI for Science applications.
tags: 
categories: 
---

<div class="intro">
This blog post discusses the design of an online deep learning framework that generates synthetic data simultaneously with the training. It summarizes the ideas of papers presented at <a href="https://dl.acm.org/doi/abs/10.1145/3581784.3607083">SuperComputing</a> and <a href="https://proceedings.mlr.press/v202/meyer23b.html">ICML</a>.
<br><br>
</div>

# Bigger Models Need Bigger Datasets

The current scaling laws of deep learning indicate that not only bigger models tend to perform better, but also that they require bigger datasets to do so. Nowadays, it is common to see training on hundreds of gigabytes if not terabytes of data. To such an extent that easily accessible big datasets are now hardly enough to train large models. There are some application domains, however, for which training data can be generated synthetically, and thus dataset are virtually unlimited. For instance, one can think about using a smaller generative model to produce synthetic texts in order to train a larger one. For this post, we focus on AI for Science applications. In this context, training data can be generated by executing an external program: typically a numerical solver that takes as input physical parameters and simulates relevant quantities ruled by partial differential equations.

# Data Limitations with the Typical Training Pipeline

The training on synthetically generated data is a two-step process. First, training data are produced using an oracle and stored to disk. Second, the actual training phase loads these data back from disk and performs the forward and backward passes to update model's weights.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/traditional_training.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    <b>Typical Training Pipeline</b>: The dataset is generated beforehand, then loaded from disk. Data loading and model updates overlap.
</div>

This process presents several disadvantages:
- Because I/O is much slower than computing, **data generation is slow**. Nonetheless, this cost is paid only once. The same dataset can be used for several trainings;
- For the same reason, because data are loaded from disk, the overall **training phase may be slow**. This is generally mitigated by overlapping data loading and model updates;
- Because disk storage is expensive and limited, **the training dataset is generally reduced to hundreds of GBs**;
- Because the dataset is generated beforehand, it is rare to generate only the data that are the most useful for training. **Training does not benefit from techniques like active learning**, curriculum learning, or Bayesian inference.

> How can we alleviate these limitations and improve large model training on synthetically generated data?

# Leveraging Network Speed

One possible solution appears while considering the characteristics of hardware. On the clusters that are used both for the training of deep learning models and the running of numerical solvers, **network speed is generally two orders faster than I/O**. For instance, if we consider an SSD device with a random read speed of 200,000 IOPS and a block size of 4 KB, we will get a throughput of 0.8 GB/s. In comparison, regarding the network, InfiniBand individual signal rate can provide a throughput up to 25 GB/s. Besides, on these clusters, **CPU hours are cheaper than storage**. 

It is thus promising to move from a traditionally offline training pipeline to an online configuration, where synthetic data are generated along the training. Multiple instances of the numerical solver are executed in parallel. Generated data are not stored on disk anymore, which circumvents I/O slowness. Instead, they are directly streamed through network for training. Training thus becomes faster. Because it needs less storage it is also cheaper. And because it allows using more data for the same training budget, it is also better.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/online_training.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    <b>Online Training Pipeline</b>: Dataset is generated on-the-fly. Data are streamed for training as soon as they are generated. It circumvents expensive I/O, allowing for faster training on more diverse dataset.
</div>


# An Online Training Framework

To expose the design of such an online training framework, we introduce the following elements:
- **Clients**: Each client runs an instance of the oracle the user provides for a different set of parameters. The oracle is a program that can be written, but not necessarily, in Python. For instance, numerical solvers are often C or Fortran programs that run on multiple processes using MPI.
- **Runner**: The runner manages the executions of the clients. It triggers and monitors oracle executions on the clients. The runner serves as an orchestrator or a job scheduler. However, we avoid these terms to prevent any confusion with job schedulers commonly found on clusters (e.g. Slurm). The runner can nonetheless rely on such schedulers. 
- **Server**: A server serves data to the training loop. It receives and consumes the data generated by the clients. It typically corresponds to the dataset of the deep learning training pipeline. There can be multiple servers in case of data distributed parallelism.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/framework_elements.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    <b>Interaction between Framework's Elements</b>
</div>


There already exist some attempts to implement this kind of online training framework (e.g. [Melissa](https://joss.theoj.org/papers/10.21105/joss.05291)). However, these frameworks require advance HPC knowledge and adds a significant overlay to PyTorch, which limits their generic use and thus adoption.

# For a Smooth Integration to Pytorch

The online framework can be reduced to a data generation tool. The actual training loop of the model should be delegated to common deep learning libraries specifically designed for that (e.g. PyTorch or JAX). The framework should incur only minimal changes to the training loop. A good way to do so is by providing a dataset class that interfaces smoothly with these libraries. For instance, in the case of Pytorch, the framework should offer a class inheriting from the [`IterableDataset`](https://pytorch.org/docs/stable/data.html#torch.utils.data.IterableDataset). 

In fact, in Pytorch, we can already obtain something similar with a `DataLoader` of multiple workers in conjunction of an `IterableDataset` object that would execute the oracle instances in the `__iter__` method. This is nonetheless limited. There will be as many instances of the oracle running in parallel as the number of workers of the data loader. This number will be bounded by the number of CPUs associated to each GPU node. It does not take advantage of CPUs that would be available on other nodes of the cluster.

```python
class Dataset(IterableDataset):

    def __iter__(self):
        # Select parameters depending on the worker info
        # Call the oracle with some parameters
        yield from oracle(parameters)

dataset = Dataset()
n_cpus = ... # Number of CPUs associated to the node
dataloader = DataLoader(dataset, num_workers=n_cpus)
```

<!-- Add figure of partitions CPUs GPUs -->

Moreover, whenever we want to guarantee _fault tolerance_, as we expect some clients to fail due to hardware failure which regularly occurs on clusters, or provide _elasticity_ for evolving resource availability, we will need finer control on the execution of the oracle. This can only be achieved by relying on a dedicated _Runner_. Several packages like [`multiprocessing.Pool`](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.pool.Pool), [`dask`](https://docs.dask.org/en/stable/), or even [`submitit`](https://github.com/facebookincubator/submitit/) can then be used for the Runner. 

# An Event-driven Architecture

By designing the framework as a set of clients, servers, and runner interacting altogether asynchronously, we are actually defining **an event-driven application**. Indeed, upon submission of new parameters by the server to the runner, the later must trigger the execution of a new oracle instance given the computational resources allocated. Upon generation of the data by the different oracle instances, they are streamed to the server, which in turns accumulate these data and yield batches. All the dynamics of the framework can be expressed as occurring events that must trigger a response. Even _fault tolerance_ feature can be expressed as a monitoring of failing oracle instances that informs the server of the issue.

Even though the event-driven architecture pattern seems fairly common and good libraries with probably similar approach exist (e.g. `dask` which uses a [`tornado.IOLoop`](https://www.tornadoweb.org/en/stable/ioloop.html)), I didn't find a clear and minimal example on how to build such architectures in Python. This kind of examples would help to avoid thread deadlocks and difficult maintenance, which are common issues for asynchronous applications.

> What is a good approach for event-driven applications in Python that are not prone to deadlocks and easy to maintain?

# A Minimal Example

In this section we present a minimal example that work locally. It only needs few modifications for running at scale on a cluster. Only the content of the `__main__` functions below would require modification in a real application.

To set up a Python environment to run the example, install the following packages. The dependencies include:
- `numpy` and `torch`;
- [`zmq`](https://pyzmq.readthedocs.io/en/latest/) for communication between runner, servers, and clients;
- `tornado` to handle asynchronous events;
- `submitit` to submit jobs to the clients.

```bash
conda create -n demo python=3.12
conda activate demo
pip install numpy torch zmq tornado submitit
# MPI Python bindings must be installed along the proper MPI binaries
CC=mpicc pip install --no-cache mpi4py 
```
<!-- Add code snippet with ZMQ as it is -->

## Utility Functions
First, we define some utility functions to format commands passed from the server to the runner and encapsulate signals sent between the different components.

<details markdown="1">
<summary markdown="span">utils.py<br><br></summary>

```python
import os
import pickle
import shlex
from dataclasses import dataclass
from enum import Enum
from functools import wraps
from typing import List

import numpy as np


@dataclass
class Task:
    """A task to execute by a client. Generated data must be send to the server address."""

    command: list[str]
    server_address: str


def format_command(command: str, *args, **kwargs) -> List[str]:
    """Format command for execution by the client."""
    full_command = (
        command
        + " ".join(args)
        + " ".join([f"{key}={val}" for key, val in kwargs.items()])
    )
    split_command = shlex.split(full_command)

    return split_command


def deserialize(msg: List[bytes]):
    return pickle.loads(msg[0])


class Status(Enum):
    """Status of running process."""

    START = -1
    SUCCESS = 0
    FAIL = 1
    READY = 2
    FINISH = 3


@dataclass
class Signal:
    """Signal sent about clients to communicate about their status."""

    status: Status
    client_id: int


@dataclass
class ClientData:
    """Numerical solvers used to generate the data
    typically produce data for different time steps."""

    data: np.ndarray
    job_id: str = None
    step_id: int = None

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(job_id={self.job_id}, step_id={self.step_id}, data_shape={self.data.shape})"


def get_rank() -> int:
    """Get the local rank for the client.
    A client may run on several processes."""
    # For Slurm
    # Get rank from environment variables
    cpus_per_task = int(os.environ.get("SLURM_CPUS_PER_TASK", 0))
    if cpus_per_task == 1:
        return 0

    rank_keys = ("RANK", "LOCAL_RANK", "SLURM_PROCID")
    for key in rank_keys:
        rank = os.environ.get(key)
        if rank is not None:
            return int(rank)

    # For MPI
    from mpi4py import MPI

    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    return int(rank)


def on_rank_zero_only(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        rank = get_rank()
        if rank == 0:
            return fn(*args, **kwargs)

    return wrapper

```

</details>


## Clients

For the client, we create a dummy example that generates random arrays for 20 time steps. In practice, the client can be any numerical simulation program. To work along the framework, the program must be edited to send the generated data over the network instead of saving them on disk, as it would be generally done. To do so, the program can use an API that is managed by the `ClientCommunicator` in the example below. This API does three things:
- initiate the communication between the client and the targeted servers (`__init__` method of the communicator);
- send data as they are produced through ZMQ socket (`send_array` method);
- close the communication and signal the runner the client has terminated successfully (`terminate` method).

Bindings to the API can be easily provided for Fortan, C, and C++ code that are generally used to write parallel numerical solvers.

<details markdown="1">
<summary markdown="span">client.py<br><br></summary>

```python
import logging
import os
import random
import socket
import time
from typing import Optional

import numpy as np
import zmq

from utils import ClientData, Signal, Status, get_rank, on_rank_zero_only

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.DEBUG)


def send_array(socket: zmq.Socket, data: ClientData):
    # TODO: Check if serializing is needed for performances
    # C.f. PyZMQ doc:
    # https://pyzmq.readthedocs.io/en/latest/howto/serialization.html#serializing-messages-with-pyzmq
    socket.send_pyobj(data)


class ClientCommunicator:
    """Class to be used by the client to send data and termination signal."""

    def __init__(self):
        self.hostname = socket.gethostbyname(socket.gethostname())
        self.rank = get_rank()
        logger.info(f"Start {self} on {self.hostname}")

        self.job_id = os.environ.get("JOB_ID", None)
        socket_addr = os.environ.get("DATA_ADDR")
        logger.debug(
            f"Instantiate communicator on rank {self.rank} of client {self.job_id}."
        )
        if self.rank == 0:
            context = zmq.Context.instance()
            self.socket = context.socket(zmq.PUSH)
            self.socket.connect(socket_addr)

    @on_rank_zero_only
    def send_array(self, data: np.ndarray, step_id: Optional[int] = None):
        client_data = ClientData(data, self.job_id, step_id)
        send_array(self.socket, client_data)

    @on_rank_zero_only
    def termniate(self):
        signal = Signal(Status.SUCCESS, self.job_id)
        self.socket.send_pyobj(signal)
        self.socket.close()


def main(steps: int = 20):
    logger.debug("Client initiates communicator.")
    communicator = ClientCommunicator()
    for step in range(steps):
        time.sleep(random.randint(0, 500) / 1_000)
        data = np.random.rand(256, 256, 2)
        communicator.send_array(data, step)
    communicator.termniate()
    logger.debug("Client terminates.")


if __name__ == "__main__":
    main()

```

</details>

## Runner

The runner starts by waiting a signal from the server for synchronization. Then it launches commands it receives from the synchronized server.

In the example, the runner relies on `submitit` to launch clients locally. `submitit` can normally be used to submit Slurm jobs. Other options exist to launch clients within an already granted Slurm allocation.

<details markdown="1">
<summary>runner.py<br><br></summary>

```python

import logging
import os
import socket
import threading
from queue import Empty, Queue
from typing import Iterator

import submitit
import zmq
import zmq.asyncio
from tornado.ioloop import IOLoop

from utils import Status, Task, get_rank

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.DEBUG)


class RunnerCommunicator:
    """Class to be used by the runner to send signals about client status."""

    def __init__(self):
        self.hostname = socket.gethostbyname(socket.gethostname())
        self.rank = get_rank()
        logger.info(f"Start {self} on {self.hostname}")
        self.servers: list[str] = []

        # Communication variables
        self.context = zmq.asyncio.Context.instance()
        self.synchronization_socket = self.context.socket(zmq.REP)
        self.synchronization_socket.bind("tcp://*:5558")
        # Event loop variables
        self._loop = IOLoop.instance()
        self._loop.add_callback(self._handle_task)
        self._loop.add_callback(self._synchronize)
        # Thread variables
        self.lock = threading.Lock()
        self._is_receiving = True
        self._tasks = Queue()
        self._thread = threading.Thread(target=self._run_ioloop)
        self._thread.daemon = True
        self._thread.start()

    @property
    def is_receiving(self) -> bool:
        with self.lock:
            return self._is_receiving

    @is_receiving.setter
    def is_receiving(self, status: bool):
        with self.lock:
            self._is_receiving = status

    def _run_ioloop(self):
        self._loop.start()

    async def _synchronize(self):
        """Synchronize with a Server"""
        while True:
            logger.debug("Waiting for synchronization...")
            (
                server_hostname,
                server_data_port,
            ) = await self.synchronization_socket.recv_pyobj()
            self.servers.append((server_hostname, server_data_port))
            self.is_receiving = True
            await self.synchronization_socket.send_pyobj(Status.SUCCESS)
            logger.debug(f"Synchronized with {server_hostname}")

    async def _handle_task(self):
        task_socket = self.context.socket(zmq.PULL)
        task_socket.bind("tcp://*:5559")
        poller = zmq.asyncio.Poller()
        poller.register(task_socket)

        timeout = -1
        while self.is_receiving:
            events = await poller.poll(timeout)
            events = dict(events)
            if task_socket in events:
                msg = await task_socket.recv_pyobj()
                if isinstance(msg, Task):
                    logger.debug(f"Receive task {msg}")
                    self._tasks.put_nowait(msg)
                # TODO: Put message in queue to avoid setting self.is_receiving
                elif isinstance(msg, Status) and msg == Status.FINISH:
                    logger.debug("Receive submission termination signal")
                    timeout = 1
            else:
                self.is_receiving = False
                self._loop.stop()

    def tasks(self) -> Iterator[Task]:
        """Iterate over the tasks received from a server submission."""
        while self.is_receiving or self._tasks.qsize():
            # TODO: Retrieve termination signal from queue
            try:
                yield self._tasks.get_nowait()
            except Empty:
                continue

    def terminate(self):
        with zmq.Context() as ctx:
            for server in self.servers:
                hostname, port = server
                with ctx.socket(zmq.PUSH) as socket:
                    socket.connect(f"tcp://{hostname}:{port}")
                    socket.send_pyobj(Status.SUCCESS)
        self._loop.stop()


if __name__ == "__main__":
    executor = submitit.AutoExecutor(folder="log_test")
    jobs = []
    communicator = RunnerCommunicator()
    for task_id, task in enumerate(communicator.tasks()):
        env = os.environ.copy()
        env.update({"DATA_ADDR": task.server_address, "JOB_ID": str(task_id)})
        function = submitit.helpers.CommandFunction(task.command, env=env)
        job = executor.submit(function)
        jobs.append(job)
    print(f"Results for {len(jobs)} jobs:")
    for job in submitit.helpers.as_completed(jobs):
        print(job.result())
    communicator.terminate()
```
</details>


## Server

The server first synchronizes with the runner. Then, it submits command to be run by the clients to generate the synthetic data. Finally, it receives these data from the clients. Whenever the clients have completed, the runner signals it to the server for it to know no more data are to expect.

<details markdown="1">
<summary>server.py<br><br></summary>

```python
import argparse
import asyncio
import logging
import os
import socket
import threading
from queue import Empty, Queue
from typing import List

import zmq
import zmq.asyncio
from tornado.ioloop import IOLoop
from torch.utils.data import IterableDataset, DataLoader

from utils import ClientData, Status, Task, format_command, get_rank
import client as dummy_client

logger = logging.getLogger(__name__)

# Set static port number for communication purposes
SYNC_PORT = 5558
TASK_PORT = 5559


class ServerCommunicator:
    """Class to be used by the server to receive and distribute client data."""

    def __init__(self, runner_hostname: str):
        self.hostname = socket.gethostbyname(socket.gethostname())
        self.rank = get_rank()
        logger.info(f"Start {self} on {self.hostname}")
        # Communication variables
        self.runner_hostname = runner_hostname
        self.context = zmq.asyncio.Context.instance()
        self.data_port = str(5560 + self.rank)
        self.task_socket = self.context.socket(zmq.PUSH)
        self.task_socket.connect(f"tcp://{self.runner_hostname}:{TASK_PORT}")
        # Event loop variables
        self._synchronized = asyncio.Event()
        self._loop = IOLoop.current()
        self._loop.add_callback(self._synchronize)
        self._loop.add_callback(self._receive)
        # Thread variables
        self._data = Queue()
        self._is_receiving = True
        self.lock = threading.Lock()
        self._thread = threading.Thread(target=self._run_ioloop)
        self._thread.daemon = True
        self._thread.start()

    @property
    def is_receiving(self) -> bool:
        with self.lock:
            return self._is_receiving

    @is_receiving.setter
    def is_receiving(self, status: bool):
        with self.lock:
            self._is_receiving = status

    async def _synchronize(self):
        """Synchronize with the runner."""
        logger.debug(f"Waiting for synchronization...")
        with self.context.socket(zmq.REQ) as runner_socket:
            runner_socket.connect(f"tcp://{self.runner_hostname}:{SYNC_PORT}")
            await runner_socket.send_pyobj((self.hostname, self.data_port))
            synchronization_status: Status = await runner_socket.recv_pyobj()
            assert synchronization_status == Status.SUCCESS
            self._synchronized.set()
            self.is_receiving = True
            logger.debug(f"Syncrhonized with {self.runner_hostname}")

    def _run_ioloop(self):
        """Method to start the tornado.IOLoop in a thread."""
        self._loop.start()

    def submit(self, command: List[str]):
        """Submit the command as a task to the runner."""
        task = Task(
            command=command, server_address=f"tcp://{self.hostname}:{self.data_port}"
        )
        self._loop.add_callback(self._submit, task)

    async def _submit(self, task: Task):
        await self._synchronized.wait()
        logger.debug(f"Submit task {task}")
        await self.task_socket.send_pyobj(task)

    async def _terminate(self):
        await self.task_socket.send_pyobj(Status.FINISH)

    def terminate(self):
        """Send signal to runner no more task will be submitted for the round.
        Delay the call to not shortcut current submissions.

        """
        self._loop.call_later(2, self._terminate)

    async def _receive(self):
        """Reception to be run in a thread.
        Listen to the sockets connected to the clients.

        """
        client_data_socket = self.context.socket(zmq.PULL)
        client_data_socket.bind(f"tcp://*:{self.data_port}")
        poller = zmq.asyncio.Poller()
        poller.register(client_data_socket)

        timeout = -1
        while self.is_receiving:
            events = await poller.poll(timeout)
            events = dict(events)
            if client_data_socket in events:
                msg = await client_data_socket.recv_pyobj()
                logger.debug(f"Received client data: {msg}")
                if isinstance(msg, ClientData):
                    self._data.put_nowait(msg.data)
                elif isinstance(msg, Status):
                    logger.debug("Wait for last messages")
                    timeout = 1
            else:
                logger.debug("Done receiving")
                self.is_receiving = False
                self._loop.stop()

    def __iter__(self):
        while self.is_receiving:
            try:
                yield self._data.get_nowait()
            except Empty:
                continue


class Dataset(IterableDataset):
    """Online dataset."""

    def __init__(self, communicator: ServerCommunicator):
        super().__init__()
        self.communicator = communicator

    def __iter__(self):
        yield from self.communicator


def main(runner_hostname: str, nsim: int, batch_size: int):
    communicator = ServerCommunicator(runner_hostname)
    for _ in range(nsim):
        # Format the command to send to the runner for execution
        # The command could include specific parameters
        command = format_command(f"python {os.path.abspath(dummy_client.__file__)}")
        communicator.submit(command)
    # Signal no more simulation will be submitted
    communicator.terminate()

    # Use the data as in a regular PyTorch training loop
    dataset = Dataset(communicator)
    dataloader = DataLoader(dataset, batch_size=batch_size, num_workers=0)
    for batch, data in enumerate(dataloader):
        print(batch, data.shape)
        # Perform the training forward and backward passes
        # ...


if __name__ == "__main__":
    parser = argparse.ArgumentParser("Multiprocessing example")
    parser.add_argument("hostname", type=str)
    parser.add_argument(
        "--nsim", type=int, default=1, help="Number of simulations to run."
    )
    parser.add_argument("--batch_size", type=int, default=1)
    args = parser.parse_args()
    runner_hostname = args.hostname
    nsim = args.nsim
    batch_size = args.batch_size
    main(runner_hostname, nsim, batch_size)
```
</details>

## Run the Example

Copy the python files above in the same folder. In one terminal run the following command to start the runner. The output will display the runner's hostname.
```bash
python runner.py
```

In another terminal, starts the server by executing the following command. For synchronization the runner's hostname must be specified (for instance `127.0.1.1`). 

```bash
python server.py <runner's hostname> --nsim 20 --batch_size 4
```

The server will ask the runner to submit jobs to the clients. The clients will then produce data and stream them to the server, These data will be available at the server level for the training loop as a regular iterable dataset.

# Could it be Implemented with Ray?

Instead of reinventing the wheel and implement the framework from scratch, we may first wonder whether there is not already existing libraries that would do the job. Indeed, such online learning settings are not new and even common in domains like _reinforcement learning_. The submodule of the library [Ray](https://docs.ray.io/en/latest/index.html) for reinforcement learning, [RLlib](https://docs.ray.io/en/latest/rllib/index.html), allows orchestrating _Actors_ that will explore the environment given a policy and send _state_ and _rewards_ to _Learners_ that will update the policy. Both _Actors_ and _Learners_ run on different resources of a cluster. We can draw a clear parallel between the use case of RLlib and our framework design by identifying _Actors_ with _Clients_, _Learners_ with _Servers_, and _rewards_ with _simulated data_ by the oracle. The comparison is nonetheless limited. With RLlib, _Actors_ are generally Python executables that run on a single process, whereas in our case the oracle is most likely to be a non-Python program that runs on multiple processes using MPI. It is not clear yet how RLlib can be tweaked to support the proposed framework.

# Conclusion

- We presented an online deep learning framework tailored for AI for Science applications;
- We provided a starting example of such framework;
- Wait a minute! You said better, cheaper, and faster? By alleviating IO, the framework accelerates data generation thus training. By reducing disk storage use it also makes it cheaper. Enabling greater data diversity improves training quality. Check the [paper](../../../assets/pdf/High_Throughput_Training_of_Deep_Surrogates_from_Large_Ensemble_Runs.pdf) for more details.

# Correspondence

Any question or suggestion to improve the framework design is welcome. You can contact me through [LinkedIn](https://www.linkedin.com/in/lucas-meyer-a7983b103/).