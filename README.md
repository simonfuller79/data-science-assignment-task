# Simon Fuller Data Task Solution

See README.rf for this and more theoretical information.

## Additional submission instructions.

The data is separate from the PR except for .pkl files.
Please read below. It is stated also in PR.

Sarah will receive these versions of the data folder:
- `upday_data_task_no_large_embedding_data.tar.bz2`
- `upday_data_task_data.tar.bz2`
These can replace the `/data` folder (i.e. they also contain the `.pkl` files.

One contains the full word embedding needed for `edaFeature.py` and `trainingModel.py`
The other contains just the subset word vector `subset_wv.txt`. The Docker image uses this by default. 

Sorry if this seems confusing but my intention here is just to simplify things for the team - i.e. the engineering aspect (flask/docker) can be assessed more quickly without the hassle of the large embedding, whilst for data science checks the larger file can be used. 
It matches my own debugging process, both here and in general, using a smaller toy model to speed up production debugging. 



Note I write in vim-tmux scripting over an REPL, see explanation in edaFeatures.py repeated at end of this file

## Docker Stuff:

-Dockerfile

-environment.yml
	- required modules for predictor.py

-DockerBuildRunInstructions
	- basic instructions to build/run image




## Python Files:

-edaFeatures.py 
	- EDA, feature examination, model selection.
	- Conclusion: comments on overall approach, repeated here.
	- I use pretrained word2vec embeddings to aggregate tokens in documents and apply ensmble learner on the continuous features.

-trainingModel.py
	- trains the model and writes a pkl
		- 'trained_rf_model.pkl'

-predictor.py
	- predicts using model.pkl
	- exposed in the docker image
	- fiddle with this and the defaults to change embedding model etc... 
- flaskPredictor.py
	- predictor.py converted to read from flask pi
	- See CONSTANTS for changing which word vector to use. If not using the full word vector it can be excluded for speed of build

-pythonSetup.py
	- script equivalent to my anaconda environment. (I think this is correct - apologies if not.)
- funs.py
	- separated functions used in predictors


## Data:

- model_vocab.pkl              
- restricted_enwiki_20180420_100d.txt
	- full word vec from wikipedia2vec
	- can be removed to speed up docker if you don't toggle the CONSTANTS in flaskPredictor to use it.
- subset_wv.txt
	- subset word vec to the training vocab. 
	- Used as default for efficiency but restricted_enwiki_20180420_100d is theoretically better since unseen words in training can still be aggregated to the embedding feature space.
- trained_rf_model.pkl
- pseudo_test_data.json        
	- json format for flask NB
- pseudo_test_data_result.tsv  
	- sample result
- pseudo_test_data.tsv         
	- psuedo test data
- upday_hungarian_data.tsv
	- a backup of the Hungarian translation due to googletrans being blocked -  just played with this in the training and eda to demo the idea. I dicuss in the relevant code.

## Conclusion of investigation and model training comments - see edaFeatures.py 
Walk through edaFeatures.py for commented steps - this just summarizes them.
All of the steps here are really just illustrative of a larger feature search I would perform.
I focused on word embeddings because it's kind of a pure approach theoretically, and also it transposes the data into a form suitable for more traditional methods like RF and XGB (as opposed to MultinomialNB or SVM we associate with count data.)
I tried 4 different parameterizations - with/without stop words, with/without URL data.Even concatenating the URL data gave a small improvement.
So no doubt generating rules, or empirically derived domain features, would further improve performance.
For instance, I would have a feature such as tfidf for certain words, like business etc..
What I like about my approach is that it lends itself more easily to multiple languages.
As we saw with the single Hungarian case (and this is why I did it) we can just machine-translate the text and then apply the embeddings approach.
So it generalizes to more languages, and to more categories, without pernickety feature engineering.
Now, I actually like, even sometimes love, pernickety feature engineering (it brings me close to the data and to the business problem) but that's not what I wanted to demonstrate today.
I would consider this model to be a kind of baseline for more empirical approaches.
-  1. It's a pretty good score, probably effective to deploy as MVP if the data matches the domain.
-  2. It's always generalizable to new categories and languages - it's robuust.
That said, google translation has become a world of pain. I remember the good old days when I was translating several million takeaway menus on ec2 just using a try/catch for when it temporarily blocked. 
I will now train the model.
I will just use the default random forest because it's so robust and quick compared to XGB.
For future steps I would add more rules/empirical features etc. (e.g. searching for certain words in text and URL (separate features for each perhaps). 
I would also consider other weightings like tfidf.
Note here I summed all repeat occurrences of words before dividing my total tokens, on an UNNORMALIZED VECTOR. Plenty of things to play about it with here:
-  1. Different word2vec parameterizations, especially cbow - given I do its aggregation basically in the mean. I used the wikipedia2vec embeddings. There are many other ones and different sources (wsj??), I just picked one more or less at random - to demonstrate the robustness of the approach as much as anything.
-  2. Different projection: in particaular I have had good experience with a cholesky decomposition (I think that's what it is or similar - been while since school). I saw a short paper on it back in 2014 for a Netflix challenge algorithm, but cannot find it now, but have used this approach on product data (product vectors). Basically I solve for wordVec * docVec' = wordCounts. 
-  ie. W = wordVec, B = wordCounts, D = docVec (or docVec^T) to find, then:

Assume: W.D = B (the document counts are the matrix product of latent word and document semantic vectors, (matrix factorizion intuition)
- W^t.W.D = W^t.B
- (W^t.W)^{-1}.(W^t.W).D = (W^t.W)^{-1}.W^t.B
- D = (W^t.W)^{-1}.W^t.B
And finally we use E=D^T, 
- i.e. a projection into the space of W, a row per document and the colmun dimension of the word vector.
 - i.e. We form a symmetric matrix out of the word vector, and use its inverse to formulate the equation in terms of D, using linear algebra e.g. np.linalg.solve() 
It's a very elegant approach. I would have liked to demonstrate it but I found an underlying bug in my gensim functions I use to reduce the word vecs per vocabs. I will redo this myself when I get a chance - I'm going to just rip out the vectors and use a named index, dispense with the gensim models entirely, because some of their backend code and model representation has changed over the years, it happens. I'll write something that just works in np over the next days - it's more implementation independent anyway. I made a mistake in my approach there.
Although, in practice the mean sentence vectors often work just as well (seems data dependent) and they are also theoretically sound - they are the basis of cbow (the middle word is compared to the aggregate of the window words), and something quite similar happens iteratively (over pairwise across the window) in skipgram, I think it's fair to say.
- 3. GlOvE, fasttext, Bert etc... and their parameterizations (e.g. fasttext substrings...)
- 4. Tf-Idf weightings, summation of normalized etc..
- 5. Training specific embeddings when the data is large enough (probably not needed - unless using supervised fasttext where we train targeted at the categories themselves.
But basically I like the purity of this approach.

Other steps like stemming etc might help but I think they might lose information, e.g. I expect more latinate word endings, ization etc, in scientific articles, less in tabloid gossip columns, etc... by aggregating through the word embedding we capture such semantic structure (is I think a well grounded theory. :) )

I would be optimistic that ifidf might lead to improvement, or anything that accounts for the global or prior probability of a given word. I've seen examples of tdidf embedding weighting around, it's a nice idea for certain tasks. The clue it might work here is that stop word removal helped already, since it kind of denoised the vectors - it's a guess but the improvement was to be expected, and I wanted to demonstrate it, the noise effect of stopwords is quite intuitive when we imagine them being components of a mean. This is probably where I would continue to explore after adding the cholesky part, since I want that code anyway. 

Note on experience, I tried such weighted product vectors at So1, e.g. downcounting for more common products like bread or beer. But there it was not so useful, because we were very interested in what common products people liked - we offered promotions on them also.

But for a classification task such as this it's more likely to have a major benefit.

Also I just used CountVectorizer() for implicit tokenization and cleaning. It is easy to use in testing also.

In prediction I allow a shortcut to load a restricted word vector, to only the words in the training set. But note, this is an unneccessary restriction. A testing case could have many words SEMANTICALLY LIKE the words in training and we would get a similar vector after aggregation. So bear in mind the restriction is more for practical loading. 

It's quite easy to have the model listening on a server on an ec2 with the larger model loaded into memory, we used that approach for years at So1 and it worked just fine, for our user load anyway...

More could be done here, but the resultant form makes it suitable for efficient word vector aggregation.

I will build a training model using no stop words with the combined text, title, url, and use the default random forest. I like it's so simple and does quite well.


## Note on the files:
I work in a vim-tmux environment, so I write here with an REPL underneath. 
e.g. as explained here:https://towardsdatascience.com/getting-started-with-vim-and-tmux-for-python-707ec5ff747f
Once you have the vim shortcuts down this leaves Jupyter in the dust.
One drawback is that it is not possible to print the output of the interpreter back into the text file.
With the N-vimR plug-in I can do this for R - it makes data evaluation safe, smooth and easy.
Here it's easy enough to copy output back with tmux shortcuts. It's not too annoying for me because I am generally taking a moment to look at the output anyway.
A really lovely advantage is how easily it is to set up on an ec2 instance when dealing with larger data sets and for large matrix multiplications etc..


##########################################################################################################################################################

# Assignment Task for Data Scientists

## Introduction
You are a Data Scientist working at upday, a news aggregator and recommender.

The engineering team at upday is gathering on a regular basis articles from all the Web. In order to provide a proper filtering functionality in the app, the articles need to be categorized.

You have at your disposal a pre-labelled dataset that maps different articles and their metadata to a specific category.

It's up to you now to help the company providing a solution for automatically categorizing articles.

## Assignment

The repository contains a dataset with some english articles and some information about them:

* category
* title
* text
* url

The purpose of the task is to provide a classification model for the articles.

## Instructions

You should make a pull request to this repository containing the solution. If you need to clarify any point of your solution, please include an explanation in the PR description.

What we expect:

* Results from your data exploration and an explanation about the solution you adopted.
* Documentation of the results of your model, including the choice of model(s), metrics adopted and the final evaluation.
* The training and evaluation code, ideally as separate scripts.

The solution should just perform better than random, also we expect you to use a model that is not just rules-based.

How to present the documentation and the code is up to you, whether to provide python scripts, jupyter notebooks or via a different mean. 


## Bonus

Show off your engineering skills. What steps would you take to productionize the model? 

We have listed some ideas below, but we would love to see the steps you would take. 
* Wrap your model into an API and document your instructions on how to test it
* Or.. Create a Docker image (or surprise us!) that can be used to run your code
* Or.. Include some unit tests in your training code
