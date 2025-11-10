# Midnight Scavenger CLI Miner

This repository contains my fork of [https://github.com/mpizenberg/ce-ashmaize/tree/piz/cli_hunt](https://github.com/mpizenberg/ce-ashmaize/tree/piz/cli_hunt).

This project provides a command-line interface (CLI) alternative to the web app for the Midnight Scavenger mining process.
Wallet creation and scavenger hunt registration are NOT handled and still require you to use the web interface, for obvious non-gaming reasons.
This tool only streamlines the continuous process of requesting, solving, and submitting new challenges, in an alternative way to the browser.
The tool can manage multiple mining addresses and keep track of their challenge progress locally.

<img alt="Screenshot of the miner" src="https://raw.githubusercontent.com/HeptaSean/midnight-miner-piz/refs/heads/hepta/cli_hunt/screenshot.png" />

## Features

*   **Challenge Management**: Automatically fetches new challenges and submits solutions.
*   **Local Data Storage**: All challenge data is stored locally in `challenges.json`, allowing for easy export and reuse with other tools.
*   **Robust Data Handling**: Utilizes an append-only journal (`challenges.json.journal`) to prevent race conditions and ensure data integrity.
*   **Efficient Solver**: Leverages a high-performance Rust binary for solving mining challenges.
*   **TUI**: Provides a Text User Interface (TUI) written in Python for real-time visual feedback on mining progress.
*   **Donate**: Provides a custom web page to help you call the `donate_to` API endpoint to consolidate your mining operations.

## Getting Started

To get this tool up and running, follow these steps:

### Prerequisites

You will need the following installed on your system:

*   **Rust and Cargo**: The Rust programming language and its package manager are required to compile the solver. You can install them from [rustup.rs](https://rustup.rs/).
*   **uv**: A fast Python package installer and dependency resolver. You can install it from [astral.sh](https://docs.astral.sh/uv/).

### Cloning the Repository

First, clone the project repository to your local machine, move into the directory and switch to the `piz` branch.
This is a fork of the [original repository](https://github.com/input-output-hk/ce-ashmaize) from IOHK implementing the ashmaize mining function.
Then move into the `cli_hunt` subfolder where I added this tool.

```bash
git clone https://github.com/HeptaSean/midnight-miner-piz.git
cd midnight-miner-piz/cli_hunt
```

### Building the Rust Solver

Navigate to the `rust_solver` directory and compile the optimized release binary:

```bash
cd rust_solver
cargo build --release
cd .. # Go back to the cli_hunt dir
```

This will create an executable solver in `cli_hunt/rust_solver/target/release/`.

> Optional: by default each solver will run with 1GB of ram, on 2 CPU threads.
> You can change the number of threads per solver by changing the constant
> `const NUM_THREADS: u64 = 2;`
> in the main.rs file, then rerun `cargo build --release`.

### Python Orchestrator Setup

Navigate to the `python_orchestrator` directory and install the Python dependencies using `uv`:

```bash
cd python_orchestrator
uv sync
```

## Usage

### Initializing the Database

Before you can run the orchestrator, you need to initialize its local database with information from your mining addresses. This information is obtained by exporting JSON metadata files from the Midnight Scavenger web application for each address you wish to manage. It's likely you'll need to register and solve at least one challenge on the website for each address to enable this export.

```bash
# from cli_hunt dir
mkdir python_orchestrator/web/
```

Place all your exported JSON files into the `cli_hunt/python_orchestrator/web/` directory.

Then, from the `cli_hunt/python_orchestrator` directory, run the initialization command:

```bash
uv run main.py init web/*.json
```

This command will read the provided JSON files and populate your `challenges.json` database.

### Running the Orchestrator

Once the database is initialized, you can start the main mining loop and TUI. From the `cli_hunt/python_orchestrator` directory, execute:

```bash
uv run main.py run
```

> Optional: by default, 2 solutions will be solved in parallel.
> You can change that number with the argument `--max-solvers 3` for example.
> Remark that by default each solver will use 1GB of ram, and 2 threads.

This will launch the TUI, providing visual feedback on the progress of fetching new challenges, solving them with the Rust binary, and submitting solutions. The orchestrator will continuously manage the mining process.

### Donate your Mining Rights

The midnight scavenger hunt has a feature to help you gather all your mining rewards from multiple addresses into a single (or few) address.
This is the `donate_to` API endpoint that you can call to donate the rights of one address to another.
Details of this endpoint are available in the following blog post: https://www.midnight.gd/news/how-to-consolidate-allocations-from-multiple-addresses-for-scavenger-mine

I’ve made a very simple web page that will help you craft that request by doing the following:
- connect to the cip-30 wallet,
- list unused addresses so that you can select the one you registered and want to donate,
- provide the recipient address (must also be registered),
- ask you to sign the donation message with your wallet,
- present you with the `curl` command to copy and paste in your terminal.

This page does not do the request directly because the various firewalls and anti-ddos protections of the API make it difficult to do directly from the web page.
Instead you are provided the curl command to run yourself in the terminal, which is much more reliable.

To open the web page, you can either use the one I host on GitHub directly https://mpizenberg.github.io/ce-ashmaize/
or use the following command in your terminal inside the `donate_to/` directory.

```sh
python -m http.server
```

This will create a minimalist static server in this folder on port 8000.
You can open the local web page at http://localhost:8000/

## Project Structure

The project is divided into two main components:

*   `cli_hunt/python_orchestrator`: Contains the Python application responsible for orchestrating the entire mining process. This includes fetching challenges, managing the local database, and submitting solutions. It also hosts the interactive TUI.
*   `cli_hunt/rust_solver`: Houses the high-performance Rust binary that performs the actual cryptographic challenge-solving computation.
*   `cli_hunt/donate_to`: Contains the web page to help you with your scavenger rights donations to consolidate rewards in few addresses.

## Data Storage

All persistent data for your mining addresses and their challenges are stored in two files within the `cli_hunt/python_orchestrator/` directory:

*   `challenges.json`: The main database file containing the current state of all managed challenges.
*   `challenges.json.journal`: An append-only journal file that records all operations. This journal is regularly consolidated into `challenges.json` and helps prevent data corruption due to race conditions or unexpected shutdowns.

## Fixing Duplicated Challenges Issue

If you cloned and run this between on the 31th of October or early 1st November, there was a bug due to duplicated instances of the challenge objects shared accross addresses.
First, make sure you terminate the running miner.
Then make sure you updated the code of this repository, with a git pull or simply recloning the repository (make sure to save your `challenges.json` file).
Then I made a small script that will detect all duplicated solutions, and reset them to the "available" status.

```sh
# from inside the python_orchestrator/ folder:
uv run reset_duplicates.py

# then you can restart the miner
uv run main.py run
```

This way, if they are still in the 24h window available to solve them, the solver will try again to solve these challenges.
This way you won’t lose the latest 24h of challenges, which should be the majority of time spent with this bug.
Remark that the TUI only shows the last 15 challenges, so it will start solving challenges that are not showed in the TUI at the beginning, then catch up with those displayed in the TUI.

## Contributing

If you find things missing and want to contribute, simply create the code you need and push it on your own fork of this.
I most likely wont integrate others code by lack of time to review this.

## Disclaimer

This tool is provided "as is," without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and non-infringement. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, out of, or in connection with the software or the use or other dealings in the software.

Users are solely responsible for their actions and for complying with all applicable laws and terms of service when using this tool. The developer of this tool is not responsible for any financial losses, account issues, or any other consequences that may arise from its use.

## License

For the full license text, please see the LICENSE file in the root of the repository.
