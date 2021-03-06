---
layout: post
title: "Как получить размер массива в Aggregation Framework"
description: ""
category: mongodb
tags: [mongodb, aggregation framework, snippets]
---


В текущей версии MongoDB(*2.4*) нет оператора для получения размера массива внутри `pipeline`a. Судя по статусу [задачи в jira](https://jira.mongodb.org/browse/SERVER-4899), эта проблема должна решиться в скором времени, а пока что придется воспользоваться временным решением(aka *костылем*). 

Пусть имеется массив следующего вида: 


    > db.test.save({x:[1,2,3,4,5]})
    > db.test.find()
    { "_id" : ObjectId("51d97482167867cbe90c5d55"), "x" : [ 1, 2, 3, 4, 5 ] }
    
Суть трюка в том, что бы сначала развернуть(*unwind*) массив, а потом сгруппировать по ключу. На этапе группировки мы смождем посчитать общее количество элементов через `{$sum: 1}` и вернуть прежний вид массива через `{$push: '$x'}`:

    > db.test.aggregate([{$unwind: "$x"}])
    {
    	"result" : [
    		{
    			"_id" : ObjectId("51d97482167867cbe90c5d55"),
    			"x" : 1
    		},
    		{
    			"_id" : ObjectId("51d97482167867cbe90c5d55"),
    			"x" : 2
    		},
    		{
    			"_id" : ObjectId("51d97482167867cbe90c5d55"),
    			"x" : 3
    		},
    		{
    			"_id" : ObjectId("51d97482167867cbe90c5d55"),
    			"x" : 4
    		},
    		{
    			"_id" : ObjectId("51d97482167867cbe90c5d55"),
    			"x" : 5
    		}
    	],
    	"ok" : 1
    }
    
А теперь сгруппируем:

    > db.test.aggregate([{$unwind: "$x"}, { $group: { _id: '$_id', x:{$push:'$x'}, size: { "$sum": 1 } } }])
    {
    	"result" : [
    		{
    			"_id" : ObjectId("51d97482167867cbe90c5d55"),
    			"x" : [
    				1,
    				2,
    				3,
    				4,
    				5
    			],
    			"size" : 5
    		}
    	],
    	"ok" : 1
    }

**При больших размерах выбоки, разворачивать и сворачивать каждый массив может быть накладно.**

**UPD.** В MongoDB начиная с версии 2.6 будет доступна команда [$size](http://docs.mongodb.org/master/reference/operator/aggregation/size/).