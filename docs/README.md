# Kernel Glossary

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        workflows/  # Kernel release model and upstreaming process.
        debugging/  # General debugging tools
        ...

## Policy

1. File names are separated with dash (-), unless it's meant for 
2. Markdown files that are not for functional purposes of Mkdocs itself (e.g. index.md) must be tagged (in the YAML front matter) according to directory they reside:

    * `workflows/`: `tooling` for a specific command line tool, or `workflows` for processes.
    * `debugging/`: `debugging`
    * `concurrency/`: `concurrency`
    * `drivers/`: `drivers`

    This location-based tagging is mandatory. They could optionally have more tags.
