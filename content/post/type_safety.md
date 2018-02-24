---
title: "Templates for secure integer casting"
date: 2018-02-24T04:02:25+01:00
draft: false
tags:
  - "cpp"
  - "stl-templates"
---


Template code for safe downcasting:

	template<typename D,
		 typename T,
		 typename std::enable_if<IS_INT(D) && IS_INT(T)>::type* = nullptr,
		 typename std::enable_if<
	    (std::numeric_limits<T>::max() > std::numeric_limits<D>::max()) ||
	    (std::numeric_limits<T>::min() < std::numeric_limits<D>::min())
		     >::type* = nullptr>
	/*!
	 * \brief If there is a risk of going out of the
	 *  range of type D, return the closest value.
	 * \param from
	 * \return
	 */
	static inline D C_FCAST(T from)
	{
	    constexpr D max_val = std::numeric_limits<D>::max();
	    constexpr D min_val = std::numeric_limits<D>::min();

	    return (from > max_val)
		    ? max_val
		    : (from < min_val)
		      ? min_val
		      : from;
	}
