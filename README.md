# Numerai Fund Project

### Overview

In this project, we'll examine the performance of various assets around the [Numerai](https://numer.ai/) ecosystem. The [Numerai Funds](https://numerai.fund/) include Numerai One and Numerai Supreme, which are hedge funds that claim to invest with "market-neutral" strategies. In addition to testing those claims using a CAPM framework, we'll also examine the relationship between the hedge fund returns and the governance token, [Numeraire (NMR)](https://www.coingecko.com/en/coins/numeraire). Since NMR is an ERC-20 token on the Ethereum blockchain, we'll also incorporate the volatility of [Ether (ETH)](https://www.coingecko.com/en/coins/ethereum) into the analysis.

### Repository Structure

The data work for this project demo is contained in the R Notebook directory of this repository. On GitHub, the webpage should display the README.md file, which contains the compiled output of the R Notebook. If you wish to explore the source code locally, then you can open the numeraifund.Rmd file in RStudio and execute the code chunks to replicate the data work. Note the `output: html_notebook` line in the header of that file, which indicates that the R Markdown document is an R Notebook. 

After exploring the R Notebook and making any desired changes, you can then create a copy that will appear on GitHub. To do this, save a copy of the R Notebook and name it README.Rmd. Then, change the header line to `output: github_document`, which will switch the file from being an R Notebook to an R Markdown file that will compile into a generic [Markdown](https://www.markdownguide.org/) file (.md). This format (along with the README name) will automatically be recognized by GitHub and displayed in-browser. This will also replace the Preview button with an option to Knit the Markdown file. This knitting process will re-run all the code chunks and generate a new README.md file inside of the R Notebook folder, which will display on GitHub.