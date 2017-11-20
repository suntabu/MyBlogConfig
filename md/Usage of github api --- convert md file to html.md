

By using [Github java api](https://github.com/eclipse/egit-github), we can convert our md files to github style htmls easily.

Here is the code in [MarkdownService.java](https://github.com/eclipse/egit-github/blob/aaf575a899df8aac9f3a74fab77a6b10007b6b39/org.eclipse.egit.github.core/src/org/eclipse/egit/github/core/service/MarkdownService.java):

``` java
	/**
	 * Get HTML for given Markdown text
	 * <p>
	 * Use {@link #getRepositoryHtml(IRepositoryIdProvider, String)} if you want
	 * the Markdown scoped to a specific repository.
	 *
	 * @param text
	 * @param mode
	 * @return HTML
	 * @throws IOException
	 */
	public String getHtml(final String text, final String mode)
			throws IOException {
		return readStream(getStream(text, mode));
	}

```