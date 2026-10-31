# Guest speaker: Will Manning, Spiral

Flash talk at the end of [lecture 14](README.md#guest-talk-spiraldb-and-vortex), starting at [61:25](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3685s). It's a shortened version of a longer talk he gave at the CMU database seminar a week earlier.

## The speaker

- Will Manning, CEO and co-founder of Spiral. The transcript doesn't say his name; it comes from Spiral's [launch post](https://spiraldb.com/blog/announcing-spiral), which he wrote in the first person.
- Before Spiral, he was one of the key people behind **Foundry** at Palantir, which he calls his first database project at scale.

## The company

- **Spiral** (spiraldb.com) builds a database for "machine consumption": queries whose results are huge and go to machines, such as training data streamed into GPUs, rather than to people.
- It's a columnar store built for object storage like S3, with primary keys and indexes, and supports PyTorch, Polars and DuckDB.
- It raised $22M in Seed and Series A funding from Amplify Partners and General Catalyst (per the launch post, Sep 2025).
- The company recently pivoted its website toward multimodal and robotics training data ("query everything... straight from object storage").
- **Vortex** is Spiral's open-source columnar file format, pitched as a faster replacement for Parquet. They donated it to the Linux Foundation, and Microsoft, Snowflake and Palantir back it.

## Main ideas from the talk

- **Three eras of data:** human-scale input and output (Postgres), then machine-scale input with human-scale output (Hadoop, data lakes, lakehouses), and now machine-scale output (feeding GPUs).
- **Network cards now have more bandwidth than a local SSD,** so the party trick works: a query over S3 can feed a GPU faster than reading Parquet from local disk. The file format decides whether you can actually reach that speed.
- **Vortex claims:** over 100x faster random access than Parquet, about 5-18x faster writes and scans, and a scan operator that can run on the CPU or directly on the GPU.
- **CPU vs GPU:** the ideas are the same; GPUs mainly need each operator's output size known in advance. "There are no new ideas in databases, but the hardware keeps changing."
- Spiral is hiring and employs a recent CMU grad (Connor Tsui).

## Links

- [Spiral website](https://spiraldb.com/)
- [Announcing Spiral](https://spiraldb.com/blog/announcing-spiral) by Will Manning, the launch post with the "Third Age of Data" argument
- [Spiral blog](https://spiraldb.com/blog), mostly deep dives on Vortex and compression
- [Vortex website](https://vortex.dev/) and [GitHub repo](https://github.com/vortex-data/vortex)
- [Vortex: a Linux Foundation Project](https://spiraldb.com/blog/vortex-a-linux-foundation-project)
- [Spiral on X](https://x.com/SpiralDB) and [jobs](https://jobs.ashbyhq.com/spiral)
- Papers he mentions are in the lecture's [Papers section](README.md#papers).
