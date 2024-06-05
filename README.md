VeriFy-Otp

const Otp = require("../../models/Otp")
const Token = require("../../models/Token")
const CustomError = require("../../utils/errors/CustomError")
const jwt = require("jsonwebtoken")

const verifyOtp = async (req, res, next) => {
    console.log(req.user, req.token)
    const otp = req.body
    const tokenData = req.token
    const otpData = await Otp.findOne({ userid: req.user._id, otp: otp, type: "OTP" });
    if (!otpData) {
        return next(new CustomError("Inccorect Otp", 401));
    }
    await Otp.findByIdAndDelete({_id:otpData._id})
    await Token.findByIdAndDelete({_id:tokenData._id})
    const Logintoken = jwt.sign({ _id: req.user._id }, process.env.JWT_SECRET_KEY, {
        expiresIn: "20m",
    });
    const tokenObject = new Token({
        userId: req.user._id,
        type: "ACCESS_TOKEN", 
        token: Logintoken
    })
    await tokenObject.save();
    return tokenObject;
}

module.exports = verifyOtp

--------------------------------------
Login Page

const User = require("../../models/User");
const bcrypt = require("bcrypt");
const { validationResult } = require("express-validator");
const jwt = require("jsonwebtoken");
const CustomError = require("../../utils/errors/CustomError");
const Otp = require("../../models/Otp")
const Token = require("../../models/Token")
const transporter = require("../../helper/sendOtp")

//User login
const userLogin = async (req, res, next) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email: email }).select([
      "password",
      "attempts",
      "lockeTime",
    ]);
    if (!user) {
      return next(new CustomError("User Not Register", 404));
    }
    const { lockeTime, attempts } = user;
    if (lockeTime > Date.now()) {
      const remaningTime = Math.ceil((lockeTime - new Date()) / 60000);
      return next(
        new CustomError(
          `Your Account Has been Lock. Please try again after ${remaningTime} minutes.`,
          403
        )
      );
    }

    const match = bcrypt.compareSync(password, user.password);
    if (!match) {
      //Lock user for 1 hour after 3 unsuccessfull login attempts with wrong password
      user.attempts += 1;
      await user.save();
      if (attempts >= 2) {
        const lockeTime = new Date();
        lockeTime.setHours(lockeTime.getHours() + 1);
        await User.updateOne(
          { email },
          { $set: { attempts: 0 }, lockeTime: lockeTime }
        );
        return next(new CustomError("your Account lock for 1 hr", 401));
      }
      return next(new CustomError("Email Or Password doesn't match", 401));
    }
    user.attempts = 0;
    user.lockeTime = null;
    await user.save();
    const token = jwt.sign({ _id: user._id }, process.env.JWT_SECRET_KEY, {
      expiresIn: "20m",
    });

    const otp = (Math.floor(100000 + Math.random() * 90000)).toString();
    console.log("otp", otp);
    await Otp.findOneAndUpdate({ userId: user._id, type: "OTP" }, { otp: otp }, { new: true, upsert: true })
    await Token.findOneAndUpdate({ userId: user._id, type: "2FA_TOKEN" }, { token: token }, { new: true, upsert: true })

    await transporter.sendMail({
      from: process.env.EMAIL_USER,
      to: email,
      subject: "Your OTP",
      html: `Your Otp is ${otp}`,
    })
    return res
      .status(200)
      .json({ status: "success", message: "Otp has been send", token: token});
  } catch (error) {
    console.log(error);
    return next(new CustomError("Unable to Login", 500));
  }
};

module.exports = userLogin;
-----------------------------------------------------
TokenAuth

const jwt = require("jsonwebtoken");
const User = require("../models/User");
const Token = require("../models/Token")
const CustomError = require("../utils/errors/CustomError");

const tokenAuth = async (req, res, next) => {
  let token;
  const { authorization } = req.headers;
  console.log(authorization);
  if (authorization && authorization.startsWith("Bearer")) {
    try {
      token = authorization.split(" ")[1];
      if (!token) {
        return next(new CustomError("Unauthorized User, No Token", 404));
      }
      const { _id } = jwt.verify(token, process.env.JWT_SECRET_KEY);
      console.log("Token 1",token);
      const user = await User.findById({_id});
      if (!user) {
        return next(new CustomError("Unauthorized User, No user Found", 401));
      }

      const tokenData = await Token.findOne({token:token, type:"2FA_TOKEN", userId:user._id});
      console.log("Token", tokenData);
      if (!tokenData) {
        return next(new CustomError("Session Expire", 401));
      }
      req.token = tokenData;
      req.user = user;
      next();
    } catch (error) {
      return next(new CustomError("Unauthorized User, No Access", 500));
    }
  }
};
module.exports = tokenAuth;
