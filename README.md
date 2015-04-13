# jquery-ajax-cache

/// <reference path="jquery-1.7.2.js" />
/// <reference path="jquery.json-2.2.js" />
/// <reference path="cacheJS.js" />
var getStackTrace = function () {
	var obj = {};
	Error.captureStackTrace(obj, getStackTrace);
	return obj.stack;
};

var ajaxTrace = {
	_arrTraceRequest: [],

	addOrUpdateRequest: function(key) {
		var r = ajaxTrace._get(key);

		if (!r) {
			ajaxTrace._add(key);
		} else {
			ajaxTrace._update(r);
		}
	},
	_add: function(key) {
		ajaxTrace._arrTraceRequest.push({ url: key.url, data: key.data, count: 1, page: ajaxTrace._getPageDomain(), trace: getStackTrace() });
	},
	_getPageDomain: function () {
		var
			firstSlash = document.location.hash.indexOf("/", 2) + 1,
			secondSlash = document.location.hash.indexOf("/", firstSlash),
			domain = document.location.hash.substring(firstSlash, secondSlash);

		return domain;
	},
	_update: function(r) {
		r.count += 1;
	},
	_get: function(key) {
		var r = ajaxTrace._arrTraceRequest.filter(function(item, idx) {
			if (item.url === key.url && item.data === key.data) {
				return true;
			}

			return false;
		});

		if (r.length > 1) {
			throw "More than one request save on trace queue!";
		}

		return r.length === 1 ? r[0] : null;
	},
	showTable: function () {
		var r = ajaxTrace._arrTraceRequest.slice().filter(function (item, idx) {
			if (item.count > 1) {
				return true;
			}

			return false;
		});

		console.table(r);
	},
	getFile: function () {
		var r = ajaxTrace._arrTraceRequest.slice().filter(function (item, idx) {
			if (item.count > 1) {
				item.data = JSON.stringify(item.data).replace(/("|{|})/g, "");

				return true;
			}

			return false;
		});

		ajaxTrace._jsonToCsvConvertor(r, "", true);
	},
	_jsonToCsvConvertor: function(JSONData, ReportTitle, ShowLabel) {
		//If JSONData is not an object then JSON.parse will parse the JSON string in an Object
		var arrData = typeof JSONData != 'object' ? JSON.parse(JSONData) : JSONData;

		var CSV = '';
		//Set Report title in first row or line

		CSV += ReportTitle + '\r\n\n';

		//This condition will generate the Label/Header
		if (ShowLabel) {
			var row = "";

			//This loop will extract the label from 1st index of on array
			for (var index in arrData[0]) {

				//Now convert each value to string and comma-seprated
				row += index + ',';
			}

			row = row.slice(0, -1);

			//append Label row with line break
			CSV += row + '\r\n';
		}

		//1st loop is to extract each row
		for (var i = 0; i < arrData.length; i++) {
			var row = "";

			//2nd loop will extract each column and convert it in string comma-seprated
			for (var index in arrData[i]) {
				row += '"' + arrData[i][index] + '",';
			}

			row.slice(0, row.length - 1);

			//add a line break after each row
			CSV += row + '\r\n';
		}

		if (CSV == '') {
			alert("Invalid data");
			return;
		}

		//Generate a file name
		var fileName = "MyReport_";
		//this will remove the blank-spaces from the title and replace it with an underscore
		fileName += ReportTitle.replace(/ /g, "_");

		//Initialize file format you want csv or xls
		var uri = 'data:text/csv;charset=utf-8,' + escape(CSV);

		// Now the little tricky part.
		// you can use either>> window.open(uri);
		// but this will not work in some browsers
		// or you will not get the correct file extension    

		//this trick will generate a temp <a /> tag
		var link = document.createElement("a");
		link.href = uri;

		//set the visibility hidden so it will not effect on your web-layout
		link.style = "visibility:hidden";
		link.download = fileName + ".csv";

		//this part will append the anchor tag and remove it after automatic click
		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
	}
};

(function ($, cacheJS, undefined) {
	var
		cacheTTL = 60, //in seconds
		_results = null,
		_key = {},

		getAjax = function (options) {
			ajaxTrace.addOrUpdateRequest(_key);

			return $.ajax(options).done(setCache);
		},
		getCache = function (options) {
			var
				deferred = jQuery.Deferred(),
				results = $.extend({}, _results, {
					ajaxSetup: jQuery.ajaxSetup({}, options)
				});

			deferred.done(options.success);
			deferred.done(options.complete);

			if (options.async === false) {
				deferredResolve(deferred, results);
			} else {
				setTimeout(deferredResolve, 1, deferred, results);
			}


			return deferred;
		},
		deferredResolve = function (deferred, res) {
			var
				callbackContext = res.ajaxSetup.context || res.ajaxSetup,
				data = res.data,
				textStatus = res.textStatus,
				jqXHR = res.jqXHR;

			deferred.resolveWith(callbackContext, [data, textStatus, jqXHR]);
		},
		setCache = function (data, textStatus, jqXHR) {
			cacheJS.set(_key, {
				data: data,
				textStatus: textStatus,
				jqXHR: jqXHR
			}, cacheTTL);
		},
		isInCache = function () {
			_results = cacheJS.get(_key);

			return _results != undefined;
		},
		getData = function (options, useCache) {
			if (!useCache || !isInCache()) {
				//---- Get using $.ajax ------
				return getAjax(options);
			}

			//---- Get from Cache, and return a deferred -----
			return getCache(options);
		},
		buildKey = function (options) {
			return {
				url: options.url,
				data: getDataForKey(options.data)
			};
		},
		getDataForKey = function (data) {
			if (typeof (data) === "string") {
				data = $.trim(data.replace(/[\{\}']/g, ""));
			}

			if ($.isNullOrEmpty(data) || $.isEmptyObject(data)) {
				data = null;
			}

			return data;
		};

	cacheJS.use('array');

	$.ajaxCache = function (options, useCache) {
		_results = null;
		_key = buildKey(options);

		useCache = useCache || false;

		return getData(options, useCache);
	}
})(jQuery, cacheJS);
