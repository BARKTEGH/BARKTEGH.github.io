---
title: stanfordCoreNLP的使用方式
date: 2019-01-21 17:09:10
tags: 
- stanford 
- 自然语言处理
categories:
- python
- 自然语言处理
---

#使用Standford Core NLP的几种方法

stanford corenlp 2018-02-27
java环境： 1.8 64位

因为斯坦福corenlp最新版都推荐使用服务器来运行，且nltk包不在支持，直接调用会出现一些问题，建议使用服务器来使用

----
##1 使用Core nlp服务器，在来连接服务器
- 创建Core nlp服务器

	在命令行中进入corenlp文件所在位置，输入开启服务器命令

		e:
		cd E:\corenlp\stanford-corenlp-full-2018-02-27 
		java -mx4g -cp "*" edu.stanford.nlp.pipeline.StanfordCoreNLPServer -port 9000 -timeout 15000
- 使用nltk（3.3v以上）包连接服务器
	API网址：https://www.nltk.org/_modules/nltk/parse/corenlp.html
		>>> from nltk.parse import CoreNLPParser
		
		# Lexical Parser
		>>> parser = CoreNLPParser(url='http://localhost:9000')
		
		# Parse tokenized text.
		>>> list(parser.parse('What is the airspeed of an unladen swallow ?'.split()))
		[Tree('ROOT', [Tree('SBARQ', [Tree('WHNP', [Tree('WP', ['What'])]), Tree('SQ', [Tree('VBZ', ['is']), Tree('NP', [Tree('NP', [Tree('DT', ['the']), Tree('NN', ['airspeed'])]), Tree('PP', [Tree('IN', ['of']), Tree('NP', [Tree('DT', ['an']), Tree('JJ', ['unladen'])])]), Tree('S', [Tree('VP', [Tree('VB', ['swallow'])])])])]), Tree('.', ['?'])])])]
		
		# Parse raw string.
		>>> list(parser.raw_parse('What is the airspeed of an unladen swallow ?'))
		[Tree('ROOT', [Tree('SBARQ', [Tree('WHNP', [Tree('WP', ['What'])]), Tree('SQ', [Tree('VBZ', ['is']), Tree('NP', [Tree('NP', [Tree('DT', ['the']), Tree('NN', ['airspeed'])]), Tree('PP', [Tree('IN', ['of']), Tree('NP', [Tree('DT', ['an']), Tree('JJ', ['unladen'])])]), Tree('S', [Tree('VP', [Tree('VB', ['swallow'])])])])]), Tree('.', ['?'])])])]
		
		# Neural Dependency Parser
		>>> from nltk.parse.corenlp import CoreNLPDependencyParser
		>>> dep_parser = CoreNLPDependencyParser(url='http://localhost:9000')
		>>> parses = dep_parser.parse('What is the airspeed of an unladen swallow ?'.split())
		>>> [[(governor, dep, dependent) for governor, dep, dependent in parse.triples()] for parse in parses]
		[[(('What', 'WP'), 'cop', ('is', 'VBZ')), (('What', 'WP'), 'nsubj', ('airspeed', 'NN')), (('airspeed', 'NN'), 'det', ('the', 'DT')), (('airspeed', 'NN'), 'nmod', ('swallow', 'VB')), (('swallow', 'VB'), 'case', ('of', 'IN')), (('swallow', 'VB'), 'det', ('an', 'DT')), (('swallow', 'VB'), 'amod', ('unladen', 'JJ')), (('What', 'WP'), 'punct', ('?', '.'))]]
		
		
		# Tokenizer 分词
		>>> parser = CoreNLPParser(url='http://localhost:9000')
		>>> list(parser.tokenize('What is the airspeed of an unladen swallow?'))
		['What', 'is', 'the', 'airspeed', 'of', 'an', 'unladen', 'swallow', '?']
		
		# POS Tagger 词性标注
		>>> pos_tagger = CoreNLPParser(url='http://localhost:9000', tagtype='pos')
		>>> list(pos_tagger.tag('What is the airspeed of an unladen swallow ?'.split()))
		[('What', 'WP'), ('is', 'VBZ'), ('the', 'DT'), ('airspeed', 'NN'), ('of', 'IN'), ('an', 'DT'), ('unladen', 'JJ'), ('swallow', 'VB'), ('?', '.')]
		
		# NER Tagger 命名实体识别
		>>> ner_tagger = CoreNLPParser(url='http://localhost:9000', tagtype='ner')
		>>> list(ner_tagger.tag(('Rami Eid is studying at Stony Brook University in NY'.split())))
		[('Rami', 'PERSON'), ('Eid', 'PERSON'), ('is', 'O'), ('studying', 'O'), ('at', 'O'), ('Stony', 'ORGANIZATION'), ('Brook', 'ORGANIZATION'), ('University', 'ORGANIZATION'), ('in', 'O'), ('NY', 'STATE_OR_PROVINCE')]
- 或者使用from stanfordcorenlp import StanfordCoreNLP，网址为https://github.com/Lynten/stanford-corenlp
		nlp = StanfordCoreNLP('http://localhost', port=9000)
		# 常用api
		print 'Tokenize:', nlp.word_tokenize(sentence)
		print 'Part of Speech:', nlp.pos_tag(sentence)
		print 'Named Entities:', nlp.ner(sentence)
		print 'Constituency Parsing:', nlp.parse(sentence)
		print 'Dependency Parsing:', nlp.dependency_parse(sentence)
	
		# 一般的
		# annotators: tokenize, ssplit, pos, lemma, ner, parse, depparse, dcoref (See Detail)
		
		# pipelineLanguage: en, zh, ar, fr, de, es (English, Chinese, Arabic, French, German, Spanish) (See Annotator Support Detail)
	
		# outputFormat: json, xml, text
		props = {'annotators': 'tokenize, ssplit, ner, depparse', 'pipelineLanguage': 'en', 'outputFormat': 'json'}
	    ParseResult = nlp.annotate(raw_sent, properties=props)
	
	
		nlp.close（）＃别忘了关闭！后端服务器将消耗大量的memery。
## 2 不使用服务器（不推荐，以不更新）
使用nltk直接调用包
2018-12-28测试发现侧方法会出现utf8解析问题，暂时无法解决，推荐使用上面的方法。
	# java路径，需要修改
	java_path = "C:\\Program Files\\Java\\jre1.8.0_191\\bin\\java.exe"
	os.environ['JAVA_HOME'] = java_path
	os.environ["STANFORD_PARSER"] = Utils.get_project_path()+"\\lib_data\\stanford-parser-full-2018-02-27\\stanford-parser.jar"
	os.environ["STANFORD_MODELS"] = Utils.get_project_path()+"\\lib_data\\stanford-parser-full-2018-02-27\\stanford-parser-3.9.1-models.jar"
	
	self.sent_parser = stanford.StanfordParser(
	                             model_path="edu/stanford/nlp/models/lexparser/englishPCFG.ser.gz",
	                           java_options='-mx8g',
	                            encoding='utf8')  #'-mlength100'
	t1 = list(self.sent_parser.parse((senten,))